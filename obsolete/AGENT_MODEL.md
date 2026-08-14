# Agent Execution Model

## Overview

This document describes the domain model for the behavior of a single agent — the full execution pipeline from receiving input to producing a response.

The model has four core concepts:

- **Task** — a unit of work that exists independently of any agent. Created by humans (specs) or by agents (decomposition). Lives on a board, has a lifecycle (created, assigned, in progress, blocked, completed, failed). Can be assigned to an agent or pulled by one.
- **Turn** — the complete sequence from input to response. An agent works on a task by executing one or more turns.
- **Step** — a single cycle through the inner loop (retrieve → reason → act). A turn contains one or more steps.
- **Stage** — an individual processing phase within a step or the outer turn structure.

---

## Task Lifecycle

A task is a persistent, trackable unit of work that exists independently of any agent's execution.

### Creation

Tasks are created by:

- **Humans** — writing specs, submitting requests, creating work items directly
- **Agents** — decomposing complex work during the Planning stage

### Assignment

Tasks can reach an agent through:

- **Push** — a parent agent or human assigns the task to a specific agent
- **Pull** — an agent takes an unassigned task from the board based on its role and capabilities

How work is assigned (push vs. pull) is a policy decision configured per team or organization, not an architectural constraint.

### States

```text
Created → Assigned → In Progress → Completed
                  ↘               ↗
                   → Blocked → Resumed
                  ↘
                   → Escalated (returned to board with context)
                  ↘
                   → Failed
```

### Relationship to Turns

An agent works on a task by executing one or more turns. A single turn may advance the task to completion, or it may leave the task in progress for future turns. Between turns, the task's state persists on the board along with any progress, blockers, or partial results.

A task may also spawn child tasks during planning — these are separate tasks on the board, linked to the parent for tracking.

---

## Pipeline Structure

A turn has an **outer frame** (runs once) and an **inner loop** (cycles until done).

```text
TURN
├── Outer Frame (once)
│   ├── Input
│   ├── Input Gate
│   ├── Tokenization
│   ├── Classification
│   └── Planning / Decomposition
│
├── Inner Loop (one or more steps)
│   │
│   │  ┌──────────────────────────────────────────────┐
│   └──│  STEP                                        │
│      │  ├── Query Transformation                    │
│      │  ├── Retrieval Gate                          │
│      │  ├── Retrieval (dense + sparse + structured) │
│      │  ├── Retrieval Evaluation (Corrective RAG)   │──► poor results? retry step
│      │  ├── Reranking                               │    with different strategy
│      │  ├── Prompt Assembly                         │
│      │  ├── LLM Call                                │
│      │  ├── Output Parsing                          │
│      │  ├── Reflection ─────────────────────────────│──► not good enough?
│      │  │                                           │    new step
│      │  ├── Agent Communication ────────────────────│──► delegate, consult,
│      │  │   ├── Delegation (pause, wait for child)  │    escalate, or notify
│      │  │   ├── Consultation (ask, wait for answer) │
│      │  │   ├── Escalation (hand off up hierarchy)  │
│      │  │   └── Notification (fire and forget)      │
│      │  ├── Tool Execution ─────────────────────────│──► tool result?
│      │  │   ├── Execution Gate                      │    new step
│      │  │   └── Execute + capture result            │
│      │  │                                           │
│      │  └── Step complete when:                     │
│      │      • LLM produces final text (no tools)    │
│      │      • Reflection accepts the output         │
│      │      • Budget/turn limit reached             │
│      └──────────────────────────────────────────────┘
│
├── Outer Frame (once, after loop exits)
│   ├── Output Guardrails
│   ├── Response Assembly
│   ├── Memory Consolidation
│   └── Response
│
└── END TURN
```

### Why this structure matters

A simple question like "What time is the meeting?" might be one step: retrieve → reason → respond.

A complex task like "Compare our auth approach to the spec and suggest improvements" might be many steps:

1. Retrieve the spec (tool call) → LLM reasons about it
2. Retrieve the current auth code (tool call) → LLM analyzes it
3. LLM drafts a comparison → reflection says it missed the token rotation section
4. Retrieve token rotation docs (tool call) → LLM revises
5. Final text response → reflection accepts

Each of those is a step. The full sequence is one turn.

---

## Cross-Cutting Concerns

### Security Model

Security runs at three gates in the pipeline. Each gate checks different things because the information available changes between stages.

- **Input Gate** — Is this agent authorized to receive this input? Is the sender trusted? Is the request within scope for this agent's role in the hierarchy? Can reject or escalate before any work is done.
- **Retrieval Gate** — Is this agent allowed to access the data sources needed? Filters which documents, projects, or scopes the agent can search. Prevents data leaking into the prompt. Enforces per-user, per-team, and per-agent data isolation. Runs at the start of every step, not just once per turn, because each step may target different data.
- **Execution Gate** — Is this agent allowed to perform the action the LLM chose? Validates tool calls, targets, and side effects before they happen. Classifies tools by risk tier (read-only / mutating / destructive). Can reject, redact, or require human approval. Runs within every step that produces a tool call.

### Observability / Tracing

Every stage emits structured trace events. This is not a pipeline stage — it is woven through every stage.

Stored (durable):

- Structured decision records: what the agent decided, which sources it used, what tools it called, and their results
- State transitions: step outcomes, reflection verdicts, gate results
- **Turn ID → Step ID → Stage** hierarchy in trace data
- Token and cost tracking per LLM call, aggregated per step and per turn
- Per-stage latency metrics
- Correlation IDs that propagate across agent-to-agent boundaries

Ephemeral (not persisted):

- Raw LLM reasoning text — summarized into structured decisions, not stored verbatim
- Intermediate prompt content — reconstructable from inputs and retrieved chunks

All stored traces are written to immutable append-only audit logs. Anomaly detection runs against behavioral baselines.

### Prompt Injection Defense

Layered defense applied across multiple stages:

- Input sanitization and classification at the Input Gate
- Strict separation of system instructions from user/tool content in Prompt Assembly
- Output validation in Output Guardrails
- Context minimization — only provide the agent with what it needs

No single technique prevents prompt injection. Defense in depth is required. See Appendix A for specific defense techniques.

---

## Outer Frame: Entry Stages

These run once at the start of a turn.

### 1. Input

An agent receives a trigger. This could be:

- A task assigned to it or pulled from the board
- A message from a human (voice transcription, chat, spec submission)
- A message from another agent (consultation, notification)
- A system event (timer, webhook, task completion)
- A tool result from a previous turn

The input is raw and unprocessed at this point.

### 2. Input Gate

First security checkpoint. Checks:

- Is the sender authenticated and trusted?
- Is this agent the right recipient for this input?
- Does the request fall within this agent's role and scope in the hierarchy?
- Does the raw input violate any policy rules (e.g. forbidden operations)?
- Prompt injection classification on untrusted input

Can reject, escalate to a higher agent, or redirect. No retrieval or LLM work happens if this gate fails.

### 3. Tokenization

The raw input is broken into tokens for downstream processing.

This is not just LLM tokenization — it includes:

- Extracting structured parts (mentions, references, commands)
- Normalizing text
- Identifying language

### 4. Classification

Determine what kind of input this is and what the agent should do with it.

Examples:

- Direct question answerable from parametric knowledge (no retrieval needed)
- Direct question requiring knowledge retrieval
- Complex task requiring planning and decomposition
- Action request requiring tool use
- Status update requiring acknowledgment
- Delegation request for another agent
- Conversational / social

This step decides:

- Whether to enter the inner loop at all (simple acknowledgments skip it)
- Whether planning is needed
- The initial retrieval strategy
- The reasoning effort level
- Whether retrieval is required or parametric knowledge is acceptable

#### No-Retrieval Policy

Some inputs may be classified as answerable from the LLM's parametric knowledge without retrieval. This is only acceptable when:

- The question is general knowledge, not domain-specific
- The risk level is low (conversational, non-binding)
- Organizational policy permits ungrounded responses for this class of input

When retrieval is skipped, Output Guardrails (E1) cannot verify claims against sources. The response must be clearly marked as ungrounded, and the guardrails must apply stricter policy checks to compensate. For high-risk or compliance-sensitive contexts, retrieval should always be mandatory regardless of classification.

### 5. Planning / Decomposition

For complex inputs, the agent breaks the goal into tasks before entering the inner loop.

- Produces an ordered or parallel set of tasks
- Each task may have its own retrieval strategy, tools, and success criteria
- Tasks the agent will do itself feed the inner loop
- Tasks for other agents are placed on the board (delegation)
- The plan can be presented for human review before execution
- The plan is a living document — reflection within a step can revise it, adding or removing tasks

#### Reasoning Effort Calibration

Before planning (and at each step), the agent determines how much reasoning effort to apply. This is driven by a configurable policy, not a hardcoded rule. Factors:

- **Task complexity** — simple lookup vs. multi-step analysis
- **Risk level** — low-stakes chat vs. production deployment decision
- **Confidence requirements** — casual answer vs. compliance-critical response
- **Time constraints** — real-time conversation vs. async background task
- **Organizational policy** — the system operator configures the default strategy (e.g. "always max quality", "optimize cost", "balance")

The effort level influences model selection, temperature, number of reflection passes, and depth of planning. This is a policy the operator sets, not a decision the agent makes autonomously.

Patterns:

- **Plan-and-Execute** — plan upfront, execute sequentially
- **ReAct** — reason and act in an interleaved loop (planning is implicit per step)
- **DAG** — parallel subtask graph with dependencies

Simple inputs skip this stage entirely based on classification.

---

## Inner Loop: Steps

A step is one cycle of retrieve-reason-act. The inner loop repeats steps until the agent has a final response.

### What triggers a new step

- **Tool result** — the LLM called a tool, got a result, and needs to reason about it
- **Reflection rejection** — the agent evaluated its own output and decided it wasn't good enough
- **Retrieval retry** — evaluation found the retrieved context was poor, needs a different strategy
- **Plan progression** — the current subtask is done, moving to the next one

### What ends the loop

- The LLM produces a final text response with no tool calls, and reflection accepts it
- A step/turn budget limit is reached (max steps, max tokens, max cost)
- An unrecoverable error occurs
- Human escalation pauses the loop

### Step Stages

#### S1. Query Transformation

Before retrieval, transform the raw query into a form that improves retrieval quality. The original query is rarely optimal for semantic or keyword search.

The transformation strategy depends on the classification and what happened in previous steps. The first step might pass the query through unchanged; a retry step after poor retrieval should try a different transformation.

Not every step needs retrieval. If the LLM already has enough context (e.g. processing a tool result), this stage and the next three are skipped.

See Appendix A for specific transformation techniques.

#### S2. Retrieval Gate

Second security checkpoint. Before retrieval executes, check:

- Which data sources is this agent allowed to query?
- Which projects, teams, or scopes can it access?
- Are there document-level or field-level restrictions?
- Per-user and per-tenant partition enforcement

Filters the retrieval scope so that sensitive data never enters the prompt. Runs every step that involves retrieval, because different steps may target different data sources.

#### S3. Retrieval

Gather context the agent needs to respond. Hybrid, combining multiple strategies:

- **Dense search** — embedding-based semantic lookup against specs, docs, meeting history, codebase
- **Sparse search** — keyword/BM25 search for identifiers, names, exact terms
- **Structured lookup** — SQL/graph queries against project state, task status, entity relationships

Results from dense and sparse search are merged using Reciprocal Rank Fusion (RRF) or weighted combination.

Metadata filtering is applied before search — scope by project, team, time range, document type. This massively improves precision.

What gets retrieved depends on classification, query transformation, and the current subtask.

#### S4. Retrieval Evaluation

After retrieval, evaluate whether the results are actually useful before proceeding.

- Score each retrieved chunk for relevance
- Discard chunks below threshold
- If all retrieved chunks score below threshold, the agent can:
  - Retry this step with a different query transformation
  - Fall back to alternative retrieval strategies (different data sources, web search)
  - Escalate or abstain ("I don't have enough information")

This prevents the "garbage in, garbage out" failure mode where bad context causes confident wrong answers.

#### S5. Reranking

Score each surviving chunk against the original query to determine actual usefulness.

- Takes (query, chunk) pairs and scores relevance jointly — more accurate than embedding distance because it sees both texts together
- Filters down to the top-k most useful chunks for the prompt

Multiple reranking stages can be chained for progressively finer filtering. See Appendix A for specific reranking approaches.

#### S6. Prompt Assembly

Build the prompt for this step's LLM call from:

- System instructions (agent role, personality, constraints, behavioral rules)
- Retrieved context from this step with source attribution markers
- Accumulated context from previous steps in this turn
- Conversation history (within session, possibly compacted)
- Tool results from previous steps
- Tool definitions
- Current plan state (if planning is active)
- Output format and grounding instructions

Strict separation of system instructions from user-supplied and tool-supplied content for prompt injection defense.

The prompt grows across steps as tool results and retrieved context accumulate. Working memory compaction may trigger if the context window fills.

#### S7. LLM Call

Send the assembled prompt to the language model.

Model, temperature, and reasoning effort may vary by agent role and task type. Token usage and cost are tracked per step and per turn.

#### S8. Output Parsing

Parse the LLM response to extract:

- Text response
- Tool call requests (function name, arguments)
- Structured data (decisions, classifications)
- Uncertainty signals
- Citations / source references

The parser determines what happens next:

- Tool calls → proceed to Agent Communication / Execution Gate / Tool Execution → new step
- Final text → proceed to Reflection
- Both → execute tools first, then evaluate

#### S9. Reflection / Self-Critique

The agent evaluates its own output before accepting it.

- Does the response actually answer the question?
- Is it consistent with the retrieved context?
- Are there gaps, contradictions, or hallucinated claims?
- Would additional retrieval or tool use improve the answer?
- Does it satisfy the current subtask in the plan?

Outcomes:

- **Accept** — output is good, exit the inner loop
- **Revise** — loop back to LLM Call with critique notes (same step, new LLM call)
- **Retry** — needs fundamentally different context, trigger a new step
- **Adjust plan** — the plan itself was wrong, revise it and continue

Reflection depth scales with task complexity and risk. Simple factoid answers get light reflection. High-stakes decisions get deep evaluation.

#### S10. Agent Communication

After reasoning and reflection, the agent may determine it needs another agent's involvement. This is not a tool call — the target is another autonomous reasoning entity.

Four modes:

- **Delegation** — create a task and place it on the board for another agent. The delegating agent may pause its inner loop and wait for the task to complete, or continue with other work. The assigned agent picks up the task and runs its own turns independently.
- **Consultation** — ask another agent a question. The inner loop pauses until the answer arrives, then resumes in the current step with the answer added to context. The current agent stays in control. This is lightweight — no task is created.
- **Escalation** — hand the current task up the hierarchy because it exceeds this agent's scope, authority, or capability. The current turn may end entirely. The task moves back to the board with updated context about what was attempted.
- **Notification** — inform another agent without expecting a response. Fire and forget, the current step continues immediately.

Flow implications:

- Delegation and consultation pause the inner loop (like human-in-the-loop but with another agent)
- Escalation exits the inner loop and potentially ends the turn
- Notification does not interrupt the flow
- The receiving agent's response may change what tools are needed next, which is why this stage comes before Tool Execution

Information flow is filtered — the child/consulted agent only receives what it's authorized to see. Cross-agent trace IDs link the turns for observability.

#### S11. Execution Gate

Third security checkpoint. Before any tool executes, check:

- Is this agent authorized to use this tool?
- Are the arguments within allowed bounds?
- Does the action target something within scope?
- What is the risk tier of this tool? (read-only / mutating / destructive)
- Does this require human approval?

Can reject the tool call, require confirmation, present a dry-run preview, or strip arguments.

#### S12. Tool Execution

If the LLM requested tool use and the execution gate passed:

- Execute the tool in a sandboxed environment
- Enforce time and resource limits
- Capture the result
- Feed result into next step (back to S6 Prompt Assembly with the tool result added to context)

Tools include: file operations, API calls, database queries, code execution.

---

## Outer Frame: Exit Stages

These run once after the inner loop exits with a final response.

### E1. Output Guardrails

Validate the agent's final response before it reaches the caller.

- Verify claims against retrieved sources
- Verify citations support the claims made
- Check policy compliance
- Validate output schema
- Trigger abstention if grounding confidence is too low

If guardrails fail, the response can be sent back into the inner loop for revision (triggers a new step).

### E2. Response Assembly

Combine everything into a coherent response:

- Final text output
- Structured results (task updates, decisions)
- Citations with provenance
- Confidence indicators
- Step count and cost summary

### E3. Memory Consolidation

After the response is produced, update the agent's memory systems.

#### Memory Types

- **Working Memory** — the current context window. Managed by summarization and compaction when it fills. A compaction boundary preserves critical information while compressing older steps and turns.
- **Episodic Memory** — records of past interactions. "Last time this topic came up, we decided X." Stored as conversation summaries or extracted key-value facts. Retrieved by similarity to current context in future turns.
- **Semantic Memory** — the agent's accumulated knowledge base. The RAG corpus itself plus facts learned during interactions. Updated when new documents, specs, or decisions are produced.
- **Procedural Memory** — how to do things. Stored as tool schemas, workflows, learned procedures. Updated when new tools or processes are introduced.

#### Consolidation Actions

- Extract new facts from this turn
- Generate new embeddings for produced knowledge
- Emit events to other agents or systems
- Update task/project state
- Write audit log entry for the complete turn

#### Write Criteria

Not everything from a turn should become a memory. A fact is written to long-term memory only when:

- It represents a decision, conclusion, or learned preference — not intermediate reasoning
- It is not already present in the knowledge base
- It has a clear source (retrieved document, human statement, tool result) — not LLM speculation

#### Conflict Resolution

When a new fact contradicts an existing memory:

- **Source authority** — a human statement overrides an agent-derived fact; a recent document overrides an older one
- **Recency** — when authority is equal, the more recent fact wins
- **Explicit override** — a human can explicitly correct a memory, which takes highest precedence

The overridden memory is marked as superseded (not deleted) to preserve audit trail.

#### Freshness

Memories carry timestamps and can have expiry policies:

- **Time-based expiry** — memories older than a configured threshold are deprioritized in retrieval, not deleted
- **Access-based decay** — memories that are never retrieved gradually lose retrieval weight
- **Explicit invalidation** — when a source document is updated or retracted, derived memories are flagged for review

### E4. Response

The final output is delivered to the caller — human, agent, or system.

---

## Context Window Management

Across steps within a turn, the context window accumulates:

- Retrieved chunks from each step
- Tool call results from each step
- LLM reasoning from each step
- The original input and plan

This can fill up fast. Management strategies:

- **Compaction** — when context approaches the limit, summarize older steps while preserving the current step's full context. Critical information (plan state, key decisions) is protected from compaction.
- **Selective retention** — not all tool results need to stay in context. Large results can be summarized or replaced with a reference.
- **Step isolation** — each step's retrieved context can be scoped. Previous steps' chunks may be dropped if they're no longer relevant.

---

## Chunking Strategy

Before any of this pipeline runs, documents must be prepared for retrieval. Chunking quality is the highest-leverage investment in the entire system.

### Principles

- Documents must be split into chunks that are topically coherent — not arbitrary fixed-size windows
- Document structure (headings, sections, tables, code blocks) should be respected as natural boundaries
- Chunks can be stored at multiple granularities to allow fine-grained retrieval with broader context expansion
- Different content types require different chunking strategies — a document classifier at ingestion time routes to the appropriate pipeline

### Per-Content-Type Strategy

| Content Type | Chunk Size | Strategy |
|---|---|---|
| Specs / requirements | 512-1024 tokens | Structure-aware (headers, sections). Keep individual requirements atomic. Never split tables. |
| Meeting transcripts | 256-512 tokens | Topic or speaker-turn segmentation. Smaller because conversations are informationally sparse. Decisions and action items should be extracted as separate indexed entities. |
| Source code | Function/class level | AST-based parsing. Never split mid-function. Keep docstrings attached to their code. File path, language, and imports stored as metadata. |
| General docs / manuals | 512-1024 tokens | Structure-aware where possible, recursive splitting with overlap as fallback. Parent-child retrieval (chunk small for precision, expand to parent for context). |

### Failure Modes

- **Too small** (<128 tokens) — embeddings lose context, retrieval gets noisy, related information fragments across many chunks
- **Too large** (>2048 tokens) — embeddings get diluted across multiple topics, wastes token budget in the prompt, reduces retrieval precision
- **Splitting mid-structure** — breaking a function, table, requirement, or conversational exchange across chunks destroys semantic coherence

### Metadata

Every chunk carries:

- Source document, section, page
- Document type (spec, meeting notes, code, manual)
- Project / team / scope
- Creation and modification timestamps
- Access control labels

---

## Agent-to-Agent Communication

Agent communication is modeled as step stage S10 in the inner loop. See that stage for the four modes (delegation, consultation, escalation, notification) and their flow implications.

Key architectural principles:

- Delegation creates tasks on the board — the delegated agent picks them up and runs its own turns independently
- Consultation is lightweight and does not create tasks — it's a direct question/answer exchange
- Escalation returns the current task to the board with context about what was attempted
- Information flow between agents is filtered by the Retrieval Gate of the receiving agent
- Delegation follows the hierarchy: agents can delegate down or escalate up
- Cross-agent trace IDs link parent and child turns for end-to-end observability

---

## Human-in-the-Loop

Agents escalate to humans when:

- Confidence falls below threshold
- Actions are classified as high-risk (destructive, financial, external communication)
- Ambiguous or conflicting instructions are detected
- Policy boundaries require it ("never send external email without approval")

Escalation **pauses the inner loop**. The step that triggered escalation is suspended until the human responds.

Escalation produces a structured approval request with:

- Proposed action and reasoning
- Retrieved context that informed the decision
- Risk assessment
- Options: approve / reject / modify

Timeout behavior: if no human responds within deadline, the system falls back to a safe default or cancels — never proceeds unsupervised on high-risk actions.

---

## Open Questions

1. ~~What embedding model strategy — shared across all agents, or domain-specific per agent role?~~ **Resolved:** Configurable per agent, defaulting to a shared model. Most agents use the shared model so all company data lives in a comparable vector space. If a specific role needs a domain-optimized model, it can be overridden in that agent's configuration without changing the architecture.
2. ~~How does the planning stage interact with multi-agent delegation? Does the planner assign subtasks to other agents?~~ **Resolved:** Introduced Task as a first-class domain concept. Planning produces tasks. Tasks the agent does itself feed the inner loop. Tasks for others are placed on the board. Delegation, pulling, and assignment are all operations on tasks. The distinction between "my work" and "work for others" is simply who the task is assigned to, not a different concept.
3. ~~What is the compaction strategy for working memory across long-running sessions?~~ **Resolved:** Tasks naturally bound context — each task has a scoped working memory. Between turns on the same task, key state (progress, decisions, blockers) is persisted to the task itself, not the context window. Within a turn, compaction uses summarization when needed, triggered at ~80% of context capacity. The current step and task state are always preserved. External memory (episodic, semantic) further reduces compaction pressure by offloading durable knowledge. The goal is to minimize how often compaction is needed rather than making it perfect.
4. ~~Should reflection depth be configurable per agent role or adaptive based on confidence?~~ **Resolved:** Both. The agent role sets a baseline reflection depth (a compliance agent reflects deeply by default, a notification agent reflects lightly). The adaptive layer adjusts within that baseline based on the specific step's confidence and risk. This is already covered by Reasoning Effort Calibration — reflection depth is one of its outputs, not a separate decision.
5. ~~What is the right chunking granularity for different document types in our domain (specs, meeting transcripts, code)?~~ **Resolved:** Different content types need different strategies. Specs: structure-aware, 512-1024 tokens, keep requirements atomic. Meetings: topic/speaker-turn segmentation, 256-512 tokens, extract decisions separately. Code: AST-based, function/class level, never split mid-function. General docs: 512-1024 tokens, structure-aware or recursive with overlap. A document classifier at ingestion routes to the appropriate pipeline. See Chunking Strategy section.
6. ~~What is the maximum step budget per turn? Should it vary by agent role or task classification?~~ **Resolved:** Three budget dimensions: step limit, cost limit, and time limit — all configurable per agent role and per task. Simple lookups get small budgets (5-10 steps), complex multi-step work gets larger ones (50-100+). The agent can check remaining budget during Reflection (S9) and decide to wrap up cleanly, request an extension via escalation, or accept the current output. Budget exhaustion should force a best-effort response, not a silent failure.
7. ~~How are steps within a turn parallelized when the plan has independent subtasks?~~ **Resolved:** Steps within a turn are inherently sequential — each step's LLM call depends on the accumulated context from prior steps. Parallelism happens at the **task level**, not the step level. When planning produces independent tasks, those tasks can be assigned to separate agents (or separate turns of the same agent) and executed concurrently. The DAG pattern in Planning (stage 5) expresses this: independent branches run in parallel as separate tasks on the board, and a synchronization point collects results before dependent work proceeds. Within a single agent's turn, the inner loop remains sequential.

---

## Appendix A: Implementation Techniques

This appendix collects specific implementation options referenced by the core model. These are not architectural decisions — they are candidate approaches to evaluate during implementation.

### Query Transformation Techniques

- **HyDE (Hypothetical Document Embeddings)** — generate a hypothetical answer, embed that instead of the raw query. Closes the gap between short questions and long document passages.
- **Step-Back Prompting** — rewrite a specific query into a more general one to retrieve broader context.
- **Sub-question Decomposition** — break a complex query into independent sub-questions, retrieve for each separately, merge results.
- **Query Expansion** — generate multiple reformulations, retrieve for each, deduplicate.

### Reranking Approaches

- **Cross-encoder models** — score (query, document) pairs jointly. High accuracy, slower.
- **Late interaction models (e.g. ColBERT)** — token-level interaction with precomputed document representations. Good balance of quality and speed.
- **LLM-based reranking** — present candidates to an LLM and ask it to rank them. Most expensive, highest quality.
- **Multi-stage pipelines** — chain fast retrieval (~100 candidates) → lightweight reranker (~20) → precise reranker (~5).

### Chunking Strategies

- **Structure-aware chunking** — parse documents using their inherent structure: headings, sections, paragraphs, lists, tables. Never split tables or code blocks.
- **Semantic chunking** — embed each sentence, find boundaries where similarity drops. Produces topically coherent chunks.
- **Hierarchical / parent-child** — store chunks at multiple granularities (sentence, paragraph, section). Retrieve at fine-grained level, expand to parent for context.
- **Proposition-based** — decompose into atomic factual statements. Most precise but expensive.

### Prompt Injection Defense Techniques

- Input classifiers trained on known injection patterns
- Canary tokens embedded in sensitive contexts to detect leakage
- Dual-LLM pattern — privileged LLM for system instructions, unprivileged LLM for untrusted content
