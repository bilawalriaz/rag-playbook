# The RAG Playbook

Distilled engineering notes from Jason Liu's RAG series (jxnl.co) — 19 articles covering retrieval-augmented generation from foundations to production, context engineering for agents, and the contrarian case against RAG for code.

Source: https://jxnl.co/writing/ (RAG Master Series + Context Engineering + Coding Agents)

## How to use this

This is a reference document, not a tutorial. Read the checklist at the bottom first. Then jump to whichever section matches your current problem.

---

## Table of contents

1. [Core architecture](#1-core-architecture)
2. [Levels of RAG complexity](#2-levels-of-rag-complexity)
3. [Systematic improvement process](#3-systematic-improvement-process)
4. [The data flywheel](#4-the-data-flywheel)
5. [Low-hanging fruit (7 quick wins)](#5-low-hanging-fruit)
6. [Six proven strategies](#6-six-proven-strategies)
7. [Production monitoring](#7-production-monitoring)
8. [Anti-patterns](#8-anti-patterns)
9. [The only 6 evals you need](#9-the-only-6-evals-you-need)
10. [Future: reports over Q&A](#10-future-reports-over-qa)
11. [Enterprise implementation](#11-enterprise-implementation)
12. [Context engineering for agents](#12-context-engineering-for-agents)
13. [Slash commands vs subagents](#13-slash-commands-vs-subagents)
14. [Compaction as momentum](#14-compaction-as-momentum)
15. [Grep beats embeddings (for code)](#15-grep-beats-embeddings)
16. [The anti-RAG case (Cline)](#16-the-anti-rag-case)
17. [Why multi-agent systems fail (Cognition)](#17-why-multi-agent-systems-fail)
18. [Rethinking RAG architecture (Sourcegraph)](#18-rethinking-rag-architecture)
19. [Model selection is not agnostic](#19-model-selection-is-not-agnostic)
20. [Numbers and benchmarks](#20-numbers-and-benchmarks)
21. [Tools reference](#21-tools-reference)
22. [Master checklist](#22-master-checklist)

---

## 1. Core architecture

RAG enhances LLMs by giving them access to external knowledge. Five components:

- **Knowledge base** — DB, docs, knowledge graph, structured/unstructured data
- **Retrieval model** — BM25 (keyword), semantic (embeddings), or hybrid
- **LLM** — takes query + retrieved context, generates response
- **Re-ranker (optional)** — refines retrieval results before they hit the LLM
- **Query understanding (optional)** — extracts intent, dates, entities; rewrites query for better retrieval

Key principles:

- Always combine full-text search (BM25) AND semantic search. Hybrid outperforms either alone.
- Start with synthetic data before deploying to real users.
- Focus on leading metrics (retrieval precision/recall, experiments per week), not lagging ones (overall satisfaction).
- Extract and use document metadata (dates, authors, tags) for filtering.
- Implement user feedback from day one.
- Continuously experiment: embedding models, re-ranking, query rewriting.

---

## 2. Levels of RAG complexity

Master each level before moving up. Most teams jump to Level 4 and wonder why everything breaks.

### Level 1: Basics

Processing pipeline:
1. Recursively traverse file system to generate text
2. Chunk text (with `window_size` and `overlap` parameters)
3. Batch requests, send to embedding API (`asyncio.Semaphore(10)` for concurrency)
4. Store in LanceDB
5. CLI for querying

Search: embed question → search DB → return results.
Answer: pass question + results to LLM.

### Level 2: Structured processing

- Better asyncio, chunking strategies, retry mechanisms
- **Cohere re-ranker** for better ranking
- **Query expansion/rewriting**: LLM extracts structured `SearchQuery` objects (via `instructor` library with `response_model`)
- **Parallel queries**: run multiple search queries simultaneously
- **Citations**: return `MultiPartResponse` with `response`, `followups`, `sources` (chunk indices)
- **Streaming**: `instructor.Partial[MultiPartResponse]` with `stream=True`

### Level 3: Observability

Log everything (wide events):

| What to log | Why |
|---|---|
| Query rewriting | Debug bad rewrites. Found "latest" was selecting current date; fixed with few-shot examples defining "latest" as >=1 week |
| Citations (shown vs cited separately) | Know what's cited vs displayed; build training data |
| Mean cosine scores + reranker scores | Cheaply identify poorly performing queries. Score 0.2/0.1 = topic not in index |
| User-level metadata | Identify groups having bad experiences: org ID, user ID, role, signup date, device, geo, language |

Operational practice: build dashboards grouped by metadata attributes showing average scores. Review during stand-up once a week.

### Level 4: Evaluations

Two systems to evaluate independently:

**A. Search system:**
- Precision and recall at various K
- Synthetic data: select random chunks → LLM generates questions → verify search retrieves source chunk → calculate recall@5, recall@K

**B. Answering system:**
- Build dataset with actual answers (not just questions)
- Generate Q+A pairs from chunks → run RAG → LLM judges correctness
- Use thumbs up/down, NOT 5-star ratings

### Level 5: Understanding shortcomings

Cluster production queries into:
1. **Topics** — what subjects users ask about
2. **Capabilities** — what operations are needed (metadata lookup, summarization, timeline, compare-and-contrast)

### Levels 6-9 (source only has bullet-point previews)

Jason's article lists these as planned topics but never expanded them beyond one-liners:

- Level 6: Finding segments and routing; processing tables; processing images
- Level 7: Building timeline queries; adding additional metadata
- Level 8: Summarization and summary indices
- Level 9: Modeling business outcomes

These are not missing from this distillation — they don't exist in the source material.

---

## 3. Systematic improvement process

### Step 1: Start with synthetic data

Biggest mistake: spending time on generation before verifying retrieval works.

1. Create synthetic questions for each text chunk
2. Test retrieval system with those questions
3. Calculate precision/recall → establish baseline
4. Identify improvement areas from baseline

Benchmarks from Jason's experience:
- Synthetic data should achieve ~97% recall/precision (below = fundamental problem)
- On essays: full-text and embeddings performed similarly, but full-text was ~10x faster
- On repo issues: full-text got ~55% recall, embeddings got ~65% recall

### Step 2: Utilize metadata

- Extract and make searchable: date ranges, file names, ownership
- Include metadata in search indexes
- Use query understanding to extract metadata from user queries

Critical example: "What is the latest X?" — neither text search nor semantic search can answer this. You MUST perform query understanding to extract date ranges.

### Step 3: Use both full-text and vector search

- Implement both; test on YOUR use case
- Use a single database system to avoid sync issues
- Anti-pattern: one client created separate indices per project → exploding array of data sources getting in/out of sync → database outage causes data loss in one system but not another

Solution: tools that do full-text + embedding + SQL against a single data object (e.g., LanceDB).

### Step 4: Implement clear user feedback

- Add feedback mechanisms ASAP
- Make copy explicitly describe what you're measuring:
  - Bad: "How did we do?" (gets thumbs down for tone, latency, formatting)
  - Good: "Did we answer the question correctly? Yes or no."

### Step 5: Cluster and model topics

Real case study: technical documentation search company found:
- Topic cluster: queries about recently updated product feature → system retrieving outdated docs
- Capability gap: users asking for troubleshooting steps → system retrieved docs but couldn't provide actionable answers
- Fix: updated docs + step-by-step instruction extraction → higher satisfaction, fewer support requests

Process:
1. Cluster queries (unsupervised learning)
2. Talk to domain experts for a couple of weeks to define categories
3. Build few-shot classifiers for topic/capability tagging
4. Feed into Amplitude or Sentry for running stream of query types

### Step 6: Continuously monitor and experiment

- Detect distribution shifts: new org onboarding can shift query distributions (e.g., deadlines went from 2% to 80% of questions)
- Run topic modeling against thumbs up/down ratings on regular cadence
- Determine count and probability of user dissatisfaction per cluster

### Step 7: Balance latency and performance

Decision framework:
- Recall doubles + latency +20% → probably worth it
- Recall +1% + latency doubles → depends on domain
- Medical diagnostic: 1% recall improvement may be worth any latency cost
- Doc page search: increased latency may cause churn

---

## 4. The data flywheel

Nine-step loop:
1. Initial implementation
2. Synthetic data generation
3. Fast evaluations
4. Real-world data collection
5. Classification and analysis
6. System improvements
7. Production monitoring
8. User feedback integration
9. Iterate (loop back)

### Leading vs lagging metrics

Stop obsessing over lagging metrics (overall application quality). Focus on leading metrics:
- Number of retrieval experiments run per week
- Precision/recall improvements on synthetic data
- Time to run evaluation suite

Analogy: weight loss — stepping on scale (lagging) doesn't cause change; tracking workouts/diet (leading) predicts change.

### Fast evaluations

Must be blazing fast (milliseconds, not seconds per question). Take search query → find relevant chunks → check if desired chunk is in results.

Anti-pattern: "I've seen teams jump straight to end-to-end evaluations with LLM-generated responses. This is a mistake. Get your retrieval working first."

### Real-world clustering

Real-world questions are stranger than synthetic ones. Process:
1. Unsupervised learning to identify topics/patterns
2. Work with domain experts to refine and label clusters
3. Build few-shot classifiers for topic distributions

Key insight: like Google specialized into Maps, Images, Shopping — you'll need targeted solutions for different question types.

### Concept drift detection

Include an "Other" category in topic classification. Monitor its percentage over time. If it grows unexpectedly → user behavior is shifting or you're onboarding customers with different needs.

---

## 5. Low-hanging fruit

Seven quick wins, ordered by impact:

### 1. Synthetic data for baseline metrics

Take existing chunks → generate synthetic questions → verify retrieval returns source chunk. Foundation for measuring everything else.

### 2. Adding date filters

"What is the latest X?" requires date filters + prompting to extract ranges. Cost: query understanding adds ~500-700ms.

### 3. Improving feedback copy

"Did we answer your question?" NOT "Did you like our response?" Store (question, answer, satisfaction) in separate table for clustering.

### 4. Tracking cosine distance and reranking scores

Log per question: `mean_cosine_distance` and `mean_cohere_reranking_score` with request ID. Trivial to implement, identifies strengths/weaknesses per query type.

### 5. Using full-text search (BM25)

Include BM25 alongside semantic search. Exact keyword matches + conceptual similarity = better overall effectiveness.

### 6. Making text chunks look like questions

Instead of HyDE at query time (adds latency), make text chunks look like questions at ingestion time. Willing to incur ingestion costs to make search better at runtime.

### 7. Including file/document metadata in chunks

Append metadata as text in each chunk during chunking: file path, document title, author, creation date, tags/categories.

---

## 6. Six proven strategies

### Strategy 1: Data flywheel with synthetic testing

Generate >=100 diverse synthetic test cases. Focus on retrieval metrics before generation quality. Use LLMs to generate AND evaluate.

### Strategy 2: Structured query segmentation

Identify distinct query patterns/types. Track performance per segment. Prioritize by: query volume x success rate x business impact.

### Strategy 3: Specialized search indices

Create dedicated indices for different content types. Combine BM25 + semantic search. Implement specialized preprocessing per data type.

### Strategy 4: Query routing and tool selection

Implement parallel function calling for multiple tools. Measure routing precision/recall separately from retrieval precision/recall. Use structured tool descriptions + few-shot examples.

### Strategy 5: Strategic user feedback

Both explicit (thumbs up/down) and implicit (user actions). Use feedback to train better embeddings, improve re-ranking, identify needed capabilities.

### Strategy 6: Response generation and presentation

Streaming responses for perceived latency. Animated progress indicators improve perceived performance by up to 11%. Implement effective citation mechanisms.

---

## 7. Production monitoring

### Why traditional monitoring fails for AI

No exception is thrown when AI fails. Running evals on production traffic is extremely expensive (LLM-as-judge). LLM judges only catch what you already know to look for. Even OpenAI admitted: "our evals didn't catch it."

### Signals to track

**Implicit signals** (from data patterns):
- User frustration expressions ("Wait, no, you should be able to do that")
- Task failures (model says it can't do something)
- NSFW content (users trying to hack the system)
- Laziness (model not completing tasks)
- Forgetting (model losing context)

**Explicit signals** (user actions):
- Thumbs up/down
- Regeneration requests
- Search abandonment
- Code errors (for coding assistants)
- Content copying/sharing (positive)

### Practical advice

For small apps (<500 daily events): pipe every interaction into a Slack channel for manual review. You need a "constant IV of your app's data."

### The Trellis framework

Targeted Refinement of Emergent LLM Intelligence through Structured Segmentation.

Three axioms:
1. **Discretization** — convert infinite output space into mutually exclusive buckets
2. **Prioritization** — score buckets by sentiment, conversion, retention, strategic priorities
3. **Recursive refinement** — continuously find more structure within buckets

Six implementation steps:
1. Initialize output space (launch minimal MVP)
2. Cluster user interactions by specific intents
3. Convert clusters into semi-deterministic workflows
4. Prioritize workflows based on company KPIs
5. Analyze workflows for sub-intents or misclassified intents
6. Recursively apply to refine each workflow

Prioritization formula:
```
Priority = Volume x Negative Sentiment x Achievable Delta x Strategic Relevance
```

Volume alone is misleading — high traffic on something you're already good at wastes effort.

### Fixing issues — portfolio of approaches

1. Prompt changes (first and simplest)
2. Offloading to tools (route to specialized tools or more capable models)
3. RAG pipeline adjustments (modify storage, memory, retrieval)
4. Fine-tuning (use identified issues as training data)

### Key principle: self-contained, blameable infrastructure

Organize system into discrete workflows. Attribute problems to specific components. Fix without affecting entire system. Measure impact per-workflow.

---

## 8. Anti-patterns

Overarching principle: "Look at your data. Start from your user, understand what they want, work backwards. Look at your data at every step."

### Data collection and curation

**Silent encoding failures:** Medical chatbot — 21% of documents silently dropped (assumed UTF-8, actually Latin-1). Fix: monitor document counts at each pipeline stage; robust error handling.

**Irrelevant document sets:** Financial news — macroeconomic trend articles included when users only wanted industry updates. Fix: curate to include only relevant content; use metadata tagging; analyze query logs to refine filters.

### Extraction and enrichment

**Chunking too small:** Many default to ~200 characters (copying outdated tutorials). E-commerce spec sheets split so small no chunk had complete info → hallucinations in 13% of queries. Modern models handle much larger chunks.

**Keeping low-value chunks:** Copyright footers, boilerplate → noise. Inspect shortest chunks manually. Deduplicate via content hashing.

### Indexing and storage

**Naive embedding usage:** Embeddings trained for semantic similarity (synonyms) but used to compare questions vs document chunks (different forms). Fixes: query expansion, late chunking/contextual retrieval, fine-tune embeddings.

**Index staleness:** Financial news — index not refreshed for 2 weeks → returned outdated earnings data. Monitor index freshness; filter by document age.

### Retrieval

**Accepting vague queries:** "Health tips" forces broad retrieval. Fix: detect low-information queries → prompt for clarification.

**Accepting off-topic queries:** "Write a poem about unicorns" in a product comparison tool. Fix: intent classification → route to fallback.

**Using full RAG for simple lookups:** "What is my billing date?" doesn't need RAG — metadata lookup is faster/cheaper/more reliable. Fix: route common structured queries to specialized handlers.

### Evaluation mistakes

**Only evaluating retrieved documents:** "Looking for keys under the lamppost." Fix: look beyond retrieval window; evaluate sufficiency, not just relevance.

**Adding complexity without evaluation:** In >90% of cases, new complex systems performed WORSE when properly evaluated. Always implement evals BEFORE increasing complexity.

### Re-ranking problems

**Overusing boosting rules:** Financial news — boosting semiconductor content + recent articles + earnings terms → unmaintainable. Fix: minimize manual boosting; train custom cross-encoder re-ranker.

### Generation phase

**Hallucination in sensitive domains:** Medical chatbot hallucinated a drug side effect not in source material. Three-step verification:
1. Force LLM to provide inline citations
2. Validate each citation exists in retrieved documents
3. Semantically validate that citation actually supports the claim

### Metadata tagging: when it matters

~40% of clients have indexes so small that metadata tagging provides little benefit. Value increases with query diversity + data scale.

---

## 9. The only 6 evals you need

Three variables: Question (Q), Context (C), Answer (A). Six conditional relationships = exhaustive coverage.

### Tier 1: Foundation (daily development)

**Retrieval precision and recall** — fast, no LLM needed. Also MAP@K, MRR@K. These are leading indicators.

### Tier 2: Primary (core evaluation, most benchmarks)

**1. Context relevance — C|Q:** How well do retrieved chunks address the question's information needs? Bad example: question about health benefits of meditation → context about types/origins of meditation.

**2. Faithfulness/groundedness — A|C:** Does the answer restrict itself to claims verifiable from retrieved context? Subtle hallucinations are most dangerous.

**3. Answer relevance — A|Q:** How directly does the answer address the specific information need? Primary user experience metric.

### Tier 3: Advanced (monthly, major releases)

**4. Context support coverage — C|A:** Does retrieved context contain ALL info needed to support every claim in the answer?

**5. Question answerability — Q|C:** Given the context, is it possible to formulate a satisfactory answer? Connects to strategic rejection.

**6. Self-containment — Q|A:** Can the original question be inferred from the answer alone?

### Implementation

| Tier | When | How |
|---|---|---|
| 1 | Daily development | No LLM needed; fast feedback |
| 2 | Core evaluation / per-release | LLM-based evaluation |
| 3 | Monthly / major releases | Strategic decisions |

### Domain-specific emphasis

- Medical RAG → higher faithfulness (A|C)
- Customer service → better answer relevance (A|Q)
- Technical docs → stronger question answerability (Q|C)

### Debugging decision tree

- Answer seems wrong? → Check faithfulness (A|C)
- Answer seems irrelevant? → Check answer relevance (A|Q)
- Answer missing key info? → Check context relevance (C|Q) or context support (C|A)

### Tools

RAGAs, ARES, TruEra RAG Triad (LLM-based eval benchmarks). Use LLM judges sparing, primarily for binary decisions with well-defined conditions.

---

## 10. Future: reports over Q&A

RAG shifts from question-answering to report generation. The underlying process can be identical (RAG in a for-loop), but the deliverable format determines perceived value.

Value framing:
- RAG Q&A: saves time for 8 employees at $50/hr
- Generated report: costs $20K but informs a $5M budget decision

SOPs are the real product. The right report template is the valuable asset. AI should produce: "This is the objective, this is how we make the decision, here are the follow-ups." NOT a chat transcript.

Future market: a marketplace of report-generating tools (templates). Skill in selecting the right report template for your desired outcome.

Actionable: don't sell "we answer questions faster." Sell "we produce the decision-making artifact that allocates your $5M budget."

---

## 11. Enterprise implementation

### The fragmentation problem (anti-patterns)

- Domain experts building isolated chatbots — none talk to each other
- Teams jumping to fine-tuning before validating the use case exists
- Every team rebuilding logging, evaluation, deployment independently
- No clear success metrics ("make it more AI" isn't a strategy)

### Three-level investment gradient

**Level 1: Discovery through chat**
- Build capabilities as MCP servers (modular services)
- Plug into simple chatbot interface
- Use Kura for hierarchical clustering of chat data
- Goal: discover what users actually want, not what you assume

**Level 2: Identified patterns become agents**
- When 10%+ of conversations cluster on one topic → build an agent
- Implementation: code-based agent, MCP server, or Temporal workflow
- Discovery-phase MCP servers become the foundation — no migration needed

**Level 3: High-value workflows get custom UI**
- Purpose-built interfaces for proven workflows
- Built on existing MCP infrastructure
- Investment backed by real usage data

### Concrete example: compliance workflow

| Timeframe | What happened |
|---|---|
| Day 1 | Added schedule data to chatbot; users ask "Who's working today?" |
| Week 2 | Manager asks about unsigned contractor compliance → build MCP server |
| Week 4 | Manager asks to send reminders → add contact search + messaging MCP servers |
| Month 2 | Cron job via Pydantic AI sends daily reminders; agent checks schedule, identifies gaps |
| Month 4 | Custom compliance dashboard with one-click calling |

After Month 1, Kura analysis revealed compliance queries = 40% of all manager interactions.

### 6-month strategic playbook

| Months | Action |
|---|---|
| 1-2 | Build shared infrastructure. Deploy chatbot. Collect usage data. |
| 3-4 | Run chat logs through Kura. Let clustering reveal patterns. |
| 5-6 | Double down on top 3-5 workflows by `volume x success_rate x value_per_interaction` |

### Contrarian take

"If you already know what the economic value is, just build the automation. Skip the chatbots and agents and focus on the work."

---

## 12. Context engineering for agents

In agentic systems, how you structure tool responses is as important as the information they contain. Tool responses ARE prompt engineering.

### Four levels of context engineering

**Level 1: Minimal chunks** — raw text only. Agents fly blind.

**Level 2: Chunks + source metadata** — adds `source`, `page`, `id` attributes. Introduces `load_pages(source, pages)` function. Agents see document clustering patterns → strategically load full pages. Includes `<system-instruction>` in tool response. Implementation effort: an afternoon.

**Level 3: Multi-modal content** — `content_types` parameter: `["text", "table", "image", "code"]`. Simple tables → Markdown; complex tables → HTML. Images include OCR text.

**Level 4: Facets + query refinement** — aggregated metadata counts alongside results (like e-commerce faceted search). High facet count + low returned chunks = valuable info filtered out by similarity ranking.

### The similarity bias problem

Resolved/done tickets have better documentation → rank higher in similarity search. Active/open issues get filtered out of top-k. Facets reveal: "All 3 returned are Done, but 5 Open tickets exist." Agent then calls `search("API timeout", status="Open")`.

### Agent persistence changes everything

Traditional RAG: optimized for humans who make ONE query. Agents: methodical, persistent, don't get frustrated. You don't need perfect recall on query #1. Give agents enough landscape context to systematically traverse.

### Measured business impact

- 90% reduction in clarification questions
- 75% reduction in expert escalations
- 95% reduction in 504 errors
- 4x improvement in resolution times

### Design principles (from Anthropic)

- Tool descriptions = prompt engineering for agents
- Return high-signal info that informs downstream actions
- Add `response_format` parameters ("concise" vs "detailed")
- Metadata that doesn't change agent behavior = expensive noise
- Prefer separate `search()` + `filter_by_date()` over one mega-tool

### The grep-as-facets connection

```bash
$ grep -r "UserService" . --include="*.py" | cut -d: -f1 | sort | uniq -c
      6 ./user_controller.py
      4 ./auth_service.py
      3 ./models.py
```

Agent sees file distribution counts → strategically calls `read_file()` on highest-signal files. This IS faceted search.

### Immediate actions

1. Audit what your tools actually return (most improvements = better string formatting)
2. Wrap results in XML, add source metadata, include system instructions
3. Implement Level 2 in an afternoon
4. Add facets (aggregated counts by source, type, status)

---

## 13. Slash commands vs subagents

Source: Jason Liu, Context Engineering series.

### The problem: context pollution

Bad context is cheap but toxic. Loading 100k lines of test logs costs nothing computationally but destroys valuable reasoning context. A well-crafted 3k-token feature spec gets wrecked when you dump Python output and error traces on top.

### Slash command path (context pollution)

When `/run-tests` dumps 150k tokens of test output into the main thread, the agent's context becomes 91% noise. The agent continues working but with degraded reasoning because most of its context is junk.

Measured on a real coding task:
- Main thread: 169,000 tokens consumed
- Useful signal: 9% (16k tokens)
- Noise: 91% (153k tokens)

### Subagent path (context isolation)

Same diagnostic capability, different economics:
- Subagent burns tokens off-thread exploring test logs, git history, file contents
- Returns distilled findings to main thread
- Main thread: 21,000 tokens consumed
- Useful signal: 76% (16k tokens)
- Noise: 24% (5k tokens)

**8x cleaner context.** Same result. Same cost ballpark.

### The principle

Burn tokens in specialized workers, preserve focus in the main thread.

### When to use subagents

- Running tests and diagnosing failures
- Processing large data rooms (financial due diligence)
- Research synthesis across multiple domains
- Any operation that generates massive, noisy output

### Caveat

Subagents work best for read-only operations. Multi-agent systems become fragile when agents make conflicting decisions without full context. For research and data exploration, parallel subagents excel. For decision-making, keep it single-threaded.

---

## 14. Compaction as momentum

Source: Jason Liu, Context Engineering series.

### The insight

If in-context learning is gradient descent (shown in research), then compaction (conversation summarization) is momentum — it preserves the learned optimization path.

### Two experiments worth running

**Experiment 1: Compaction timing affects task success**

Run million-token agent trajectories on complex tasks. Test compaction at different completion points (50%, 75%, natural boundaries, agent-controlled). Key question: does timing affect how well agents maintain their learning trajectory?

**Experiment 2: Compaction for observability**

Use specialized compaction prompts to understand agent failure patterns:

- Failure mode detection: compact focusing on loops, linter conflicts, deleted code recreation
- Language switching: compact focusing on framework switches, polyglot patterns
- User feedback clustering: compact focusing on corrections, preference statements

### Practical implications

- Simple summarization often beats complex context management (Cline's finding)
- To-do lists help agents track progress across context resets
- Compaction timing matters more than most teams realize
- The "bitter lesson" applies: simpler compaction strategies win as models improve

---

## 15. Grep beats embeddings

Source: Colin Flaherty, founding engineer at Augment (SWE-Bench Verified leaderboard-topping agent).

### Core finding

"We explored adding various embedding-based retrieval tools, but found that for SWE-Bench tasks this was not the bottleneck — grep and find were sufficient."

Agent persistence compensated for simple tools.

### Why grep+find worked

- Repositories relatively small
- Code is highly structured with distinctive keywords
- Agent persistence compensated for less sophisticated tools
- 90% of SWE-Bench problems solvable by good engineer in <1 hour

### Advantages of agent + grep/find

- Iterative retrieval is trivially simple
- Token budget management: truncate old tool calls, rerun if needed
- Zero infrastructure: no vector DBs, no syncing
- Natural course correction: if one approach fails, try another

### Architecture decision framework

| Factor | Traditional RAG | Agent+Grep | Agent+Embeddings |
|---|---|---|---|
| Quality | Decent | Excellent | Best |
| Latency | Excellent | Poor | Poor |
| Cost | Low | High | High |
| Reliability | No course correction | Self-correcting | Self-correcting |
| Scalability | High | Low | Medium |
| Maintenance | Medium | Low | High |

### When embeddings become essential

- Large codebases (millions of files)
- Unstructured content (Slack messages, documentation)
- Third-party code models haven't memorized
- Non-text media (video recordings)

### Colin's heuristic

"If I was a really persistent human that never got tired, would having this other search tool help me? If yes, it's probably useful for the agent."

### Improving agentic retrieval

- Add re-rankers to embedding tools (improve precision, reduce tokens)
- Train specialized embedding models per task (code vs Slack)
- Prompt-tune tool schemas for efficient agent usage
- Hierarchical retrieval: summarize files/directories
- Asynchronous pre-processing: create LLM-generated "dossiers" per item at ingestion time (turned a non-working search system into one that works well)

### Evaluation: vibe-first approach

1. Start with 5-10 examples
2. Do end-to-end vibe checks (qualitative)
3. Only move to quantitative evaluation after addressing obvious issues

### Key recommendation

"Don't throw away your existing retrieval systems — expose them as tools to agents."

---

## 16. The anti-RAG case (Cline)

Source: Nik Pash, Head of AI at Cline.

### Core stance

"At Cline, I became the number one advocate against using RAG, even though RAG has been my bread and butter for so long."

Corroboration: Boris Cherny (Claude Code) said on the Latent Space podcast that Anthropic tried RAG early, then moved to agentic search because it outperformed everything else.

### Why RAG fails for code

1. **Security:** indexing entire codebase = embeddings can be reverse-engineered to recover original content
2. **Maintenance overhead:** embeddings need storage, updates, syncing
3. **Agent distraction:** even perfect chunking gives disconnected snippets
4. **Resource black hole:** optimizing RAG pipelines consumes endless resources for marginal gains

### The alternative: "plan and act"

Mimics how senior engineers explore codebases:
1. Examine folder structure and file names
2. Read files in entirety → understand imports/dependencies
3. Use grep for specific patterns
4. Build understanding through agentic discovery

"Narrative integrity" = agent follows coherent thought process vs jumping between disconnected chunks.

### When RAG still makes sense

| Scenario | Why |
|---|---|
| Cost optimization ($20/mo subscription) | Avoids loading entire files; reduces tokens |
| Perfunctory performance | Basic functionality, not high-quality intelligence |
| High-volume, low-stakes (1000s of PR reviews) | Full context processing cost unjustified |

### Context management for long tasks

- `/sum` slash command: compact conversation via summarization
- `/new task`: creates summary as if handing off to new engineer → fresh context window
- Simple summarization > complex context management strategies

### The bitter lesson for app developers

"The application layer is shrinking over time. Every day it's growing smaller and smaller."

"Just throw it all out. Let the model do its job. Stop trying to get in the way of the model."

### On multi-agent systems

"It just never worked. It was just a fast way to burn a whole bunch of tokens and get nowhere."

Recommendation: single-threaded, one agent for coding tasks.

### Model selection pattern

- Planning phase: large context window model (e.g., Gemini Pro 2.5)
- Execution phase: specialized coding model (e.g., Sonnet)

---

## 17. Why multi-agent systems fail (Cognition)

Source: Walden Yan, co-founder and CPO of Cognition (Devin).

### Core problem: the telephone game

Multi-agent systems break down because of context loss. Each agent only knows what the orchestrator told it. Critical details get lost in transmission.

Example: one agent builds green pipes (Flappy Bird background), another builds a bird asset. Without shared context, they produce incompatible components. This compounds at scale.

### Context passing helps but doesn't solve it

Even with full context passing (entire agent traces), parallel sub-agents make implicit decisions that conflict:
- Different coding styles
- Different API choices
- Duplicated code

These conflicts create integration problems that the orchestrator can't resolve without full context of both agents' reasoning.

### Linear systems hit context limits

Sequential agents (agent 1 → agent 2 → agent 3) avoid conflicts but accumulate context until it exceeds the window. Cognition trained a specialized model to identify and preserve critical information across agent handoffs.

### Real-world patterns that work

**Read-only sub-agents** (Claude Code, OpenCode):
- Sub-agents only read, never make decisions
- List files, examine packages, look for imports
- Report findings back to main agent
- Main agent retains all decision authority

**Edit-apply models** (Cursor, Windsurf):
- Smart model generates human-readable edit instructions
- Simpler model applies those changes
- Fragility: if instructions are ambiguous, edits break

### The user-facing rule

Systems should feel like a single coherent agent to users. Even with complex internals, present one continuous decision-maker. True multi-agent collaboration requires modeling what others know — a skill current LLMs lack.

---

## 18. Rethinking RAG architecture (Sourcegraph)

Source: Beyang Liu, CTO of Sourcegraph (Amp coding agent).

### The paradigm inversion

RAG chat era: monolithic context engine fetches snippets → sends to LLM → generates response.

Agentic era: model decides which tools to invoke → fetches its own context → reasons about what to explore.

This is not a minor change. It inverts who controls context fetching.

### Chat era vs agentic era

Chat era: humans deeply involved in inner loop. Check output after every LLM turn. Refine. Apply. Lots of ping-pong for one atomic change.

Agentic era: articulate what you want upfront. Agent reads files, edits, searches, executes, checks output. Much less human babysitting.

### What RAG looks like now

Traditional RAG: monolithic engine with keyword indexes, embedding models, domain-specific chunkers, re-rankers.

Modern agent: portfolio of simple Unix-like tools:
- grep and glob for basic searches
- Search sub-agent for multi-query exploration
- Web documentation tools
- Specialized services

"RAG is not strictly about retrieval anymore. It's about molding the underlying model to be able to do what you need in a particular application setting."

### Sub-agents as context extenders

Amp uses three types:
1. **Code search sub-agent** — explores and refines queries, consumes its own context window, returns only relevant snippets
2. **Generic sub-agent** — invokes main agent in parallel for independent tasks
3. **Oracle sub-agent** — uses a different model (Claude 3) better at nuanced thinking

The search sub-agent is key: it burns context on exploration, then returns compact results. You throw away the exploration context.

### Sourcegraph's controversial decisions

- Minimal UI — focus on agent design, not context selection GUIs
- Bias toward action — agent edits files without asking permission
- No model selector — intentional coupling between models and tools
- Unix philosophy — composable tools, not vertically integrated clients
- Usage-based pricing — avoid perverse incentives to dumb down agent

---

## 19. Model selection is not agnostic

Source: Beyang Liu (Sourcegraph), Nik Pash (Cline), Colin Flaherty (Augment).

### The coupling problem

Chat era: user message → retrieve context → LLM → response. Models loosely coupled with retrieval. Easy to swap.

Agent era: agent LLM uses tools, tool descriptions become part of effective prompt, some tools are agents with their own models. Tight coupling.

### Why model swapping breaks agents

Tool descriptions are tuned for specific models. If you swap to a model that hasn't been tuned for your tool schemas, the agent misuses tools, hallucinates parameters, or ignores available tools entirely.

"If we offer users a way to easily swap out any of these LLMs for another model that's not been tuned to those tool descriptions, it's a recipe for a bad user experience."

### The emerging pattern

Different models for different phases:
- Planning: large context window, strong reasoning (Gemini Pro 2.5)
- Execution: specialized coding model (Sonnet)
- Oracle tasks: model with deep nuanced thinking (Claude 3)

This is intentional coupling, not model agnosticism. The tool ecosystem is designed around specific model capabilities.

### Implication for your RAG system

Don't build model-agnostic if you're building agents. Design your tool descriptions and system prompts for the specific model you're using. Swapping models later requires re-tuning the entire tool interface, not just changing an API key.

---

## 20. Numbers and benchmarks

| Metric | Value | Context |
|---|---|---|
| Synthetic data target recall/precision | ~97% | Below = fundamental retrieval problem |
| Full-text vs embeddings (essays) | Same quality, 10x faster for full-text | Jason's experiment |
| Full-text recall (repo issues) | ~55% | Jason's experiment |
| Embedding recall (repo issues) | ~65% | Jason's experiment |
| Query understanding latency cost | 500-700ms | Adding date filters/metadata extraction |
| Docs silently dropped (encoding) | 21% | Medical chatbot, UTF-8 vs Latin-1 |
| Hallucination rate from tiny chunks | 13% | E-commerce, ~200-char chunks |
| Complex systems performing worse | >90% | When added without proper evaluation |
| Metadata tagging unnecessary | ~40% of clients | Indexes too small to benefit |
| Animated progress indicators | +11% perceived performance | UX research |
| Context engineering impact | 90% fewer clarifications, 75% fewer escalations, 95% fewer 504s, 4x faster resolution | Article 12 |
| Subagent context cleanliness | 76% signal vs 9% signal (slash) | 8x improvement, article 13 |
| Small app threshold for manual review | <500 daily events | Pipe to Slack, review all |
| Time to stability after launch | ~4 months | Continuous data review loop |
| Compliance queries in manager interactions | 40% | Discovered via Kura clustering |
| SWE-Bench problems solvable in <1 hour | 90% | By a good engineer |

---

## 21. Tools reference

| Tool | Purpose |
|---|---|
| LanceDB | Vector DB with full-text + embedding + SQL in single object |
| Cohere | Re-ranking API |
| Instructor (Python) | Structured output from LLMs (`response_model`, `Partial` for streaming) |
| Pydantic | Data validation, structured responses |
| BM25 | Full-text/keyword search |
| Amplitude | Product analytics for query type tracking |
| Sentry | Error/performance monitoring |
| Kura | Chat data clustering/analysis (inspired by Anthropic's Clio) |
| Raindrop | AI production monitoring |
| Oleve / Trellis | AI product management framework |
| Lilypad (Mirascope) | RAG evaluation with versioning best practices |
| RAGAs, ARES, TruEra RAG Triad | LLM-based RAG evaluation benchmarks |
| TurboPuffer | Vector DB with facets and aggregations |
| Extend, Reducto | Structured data extraction from documents for facets |
| Pydantic AI | Workflow automation framework |
| Temporal | Workflow orchestration |
| MCP (Model Context Protocol) | Modular service interface for AI capabilities |

---

## 22. Master checklist

### Phase 1: Foundation (week 1-2)

- [ ] Set up LanceDB (or similar) with both full-text + vector search
- [ ] Implement chunking with metadata appended (file path, title, author, date, tags)
- [ ] Use asyncio with Semaphore(10) for embedding API calls
- [ ] Batch embedding requests (batch size ~10)
- [ ] Generate synthetic questions for every text chunk (few-shot prompt)
- [ ] Run retrieval eval: verify recall@5, recall@10 → target ~97%
- [ ] If below 97%, debug retrieval BEFORE touching generation

### Phase 2: Search quality (week 2-4)

- [ ] Add Cohere re-ranking
- [ ] Implement query rewriting/expansion via LLM (Instructor library)
- [ ] Add date filter extraction (query understanding, ~500-700ms cost)
- [ ] Log mean cosine distance + Cohere reranking score per query
- [ ] Log citations (shown vs cited separately)
- [ ] Log user metadata (org, user, role, device, geo, language)
- [ ] Build dashboards grouped by metadata attributes

### Phase 3: Feedback and evaluation (week 4-6)

- [ ] Implement thumbs up/down with copy: "Did we answer the question correctly?"
- [ ] Store (question, answer, satisfaction) in separate table
- [ ] Build answering eval: generate Q+A pairs, run RAG, LLM-judge correctness
- [ ] Set up weekly stand-up review of scores + examples
- [ ] Generate at least 100 diverse synthetic test cases

### Phase 4: Production intelligence (month 2+)

- [ ] Cluster real user queries (unsupervised learning)
- [ ] Work with domain experts to label clusters (topics + capabilities)
- [ ] Build few-shot classifiers for topic/capability tagging
- [ ] Feed classifications into Amplitude/Sentry
- [ ] Add "Other" category; monitor for concept drift
- [ ] Run topic modeling against satisfaction ratings on regular cadence
- [ ] Prioritize: Volume x Negative Sentiment x Achievable Delta x Strategic Relevance

### Phase 5: Context engineering (month 2+)

- [ ] Audit tool responses: are you returning metadata or just text?
- [ ] Add XML structure + source attributes to search results
- [ ] Add `<system-instruction>` blocks teaching agents how to use results
- [ ] Implement `load_pages()` for full-document loading
- [ ] Add facets (aggregated counts by source, type, status) to search responses
- [ ] Identify operations that generate massive noisy output → route to subagents
- [ ] Measure signal/noise ratio in main thread vs subagent path
- [ ] For long-running tasks: test simple summarization vs complex context management
- [ ] If building multi-agent: restrict sub-agents to read-only operations
- [ ] Design tool descriptions tuned for your specific model (not model-agnostic)

### Phase 6: Continuous improvement (ongoing)

- [ ] Run experiments: tweak chunking, embedding models, re-ranking, query rewriting
- [ ] Measure recall + latency impact of every change
- [ ] Make latency/performance trade-off decisions based on domain
- [ ] Refine synthetic data generation based on real-world insights
- [ ] Review "Other" category growth for emerging needs
- [ ] Evaluate with the 6 metrics (Tier 1 daily, Tier 2 per-release, Tier 3 monthly)
- [ ] Fix issues via: prompt changes → tool offloading → RAG adjustments → fine-tuning

### Strategic decisions

- [ ] Consider report generation over Q&A for higher perceived value
- [ ] For code: try grep+find before building embedding infrastructure
- [ ] For agents: give peripheral vision (facets), not perfect answers
- [ ] Embrace the bitter lesson: remove application-layer complexity as models improve
- [ ] Use large-context model for planning, specialized model for execution
- [ ] For multi-agent: start with read-only sub-agents; avoid parallel decision-making agents
- [ ] For long tasks: try simple summarization first before complex context management
- [ ] Don't build model-agnostic if building agents — design tools for your specific model

---

*Compiled from 19 articles at jxnl.co/writing/ (July 2026). All credit to Jason Liu and the cited experts (Skylar Payne, Colin Flaherty, Nik Pash, Walden Yan, Beyang Liu, Ben from Raindrop, Sid from Oleve).*
