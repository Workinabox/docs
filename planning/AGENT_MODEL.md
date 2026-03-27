# Agent Execution Model

## Overview

This document describes the domain model for the behavior of a single agent — the full execution pipeline from receiving input to producing a response.

The pipeline has three layers:

- **Turn** — the complete sequence from input to response. One human message produces one turn.
- **Step** — a single cycle through the inner loop (retrieve → reason → act). A turn contains one or more steps.
- **Stage** — an individual processing phase within a step or the outer turn structure.

Stages contain individual operations annotated as either `(function)` or `(trait)`. A **function** is deterministic given its inputs. A **trait** has a pluggable implementation — it defines an interface that can be backed by an LLM, a specialized model, or a heuristic depending on configuration.


---

## Pipeline Structure

A turn has an **outer frame** (runs once) and an **inner loop** (cycles until done).

```
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

- Full trace of agent reasoning: prompt sent, model response, tool call, tool result, decision made
- Correlation IDs that propagate across agent-to-agent boundaries
- **Turn ID → Step ID → Stage** hierarchy in trace data
- Token and cost tracking per LLM call, aggregated per step and per turn
- Per-stage latency metrics
- Immutable append-only audit logs
- Anomaly detection against behavioral baselines

### Prompt Injection Defense

Layered defense applied across multiple stages:

- Input sanitization and classification at the Input Gate
- Strict separation of system instructions from user/tool content in Prompt Assembly
- Output validation in Output Guardrails
- Context minimization — only provide the agent with what it needs
- Canary tokens in sensitive contexts to detect leakage

No single technique prevents prompt injection. Defense in depth is required.

---

## Outer Frame: Entry Stages

These run once at the start of a turn.

### 1. Input
An agent receives a trigger. This could be:

- Receive trigger from human, agent, system event, or tool result (function)
- Validate message envelope / format (function)

### 2. Input Gate
First security checkpoint. Checks:

- Authenticate sender (function)
- Check agent scope and role authorization (function)
- Check policy rules (function)
- Classify prompt injection on untrusted input (trait)
- Reject, escalate, or redirect on failure (function)

No retrieval or LLM work happens if this gate fails.

### 3. Tokenization
The raw input is broken into tokens for downstream processing.

This is not just LLM tokenization — it includes:

- Extract structured parts — mentions, references, commands (function)
- Normalize text (function)
- Identify language (function)

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

Operations:

- Classify input intent (trait)
- Decide whether to enter the inner loop (function)
- Select retrieval strategy (function)
- Determine reasoning effort level (function)

### 5. Planning / Decomposition
For complex inputs, the agent breaks the goal into a structured plan of subtasks before entering the inner loop.

- Decompose goal into subtasks (trait)
- Assign retrieval strategy, tools, and success criteria per subtask (function)
- Present plan for human review if required (function)
- The plan is a living document — reflection within a step can revise the plan

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
Before retrieval, transform the query to improve retrieval quality.

- Transform the query (trait) — implementation selected based on classification and previous steps:
  - **HyDE** — generate a hypothetical answer, embed that instead of the raw query
  - **Step-Back Prompting** — rewrite into a more general query for broader context
  - **Sub-question Decomposition** — break into independent sub-questions, retrieve separately, merge
  - **Query Expansion** — generate multiple reformulations, retrieve for each, deduplicate
- Merge results from multiple sub-queries (function)

Not every step needs retrieval. If the LLM already has enough context (e.g. processing a tool result), this stage and the next three are skipped.

#### S2. Retrieval Gate
Second security checkpoint. Before retrieval executes, check:

- Check data source authorization (function)
- Apply project/team/scope filters (function)
- Enforce document-level and field-level restrictions (function)
- Enforce per-user and per-tenant partitions (function)

Filters the retrieval scope so that sensitive data never enters the prompt. Runs every step that involves retrieval, because different steps may target different data sources.

#### S3. Retrieval
Gather context the agent needs to respond. Hybrid, combining multiple strategies:

- Apply metadata filters — scope by project, team, time range, document type (function)
- Dense search — embedding-based semantic lookup (function)
- Sparse search — keyword/BM25 for identifiers, names, exact terms (function)
- Structured lookup — SQL/graph queries against project state, task status, entity relationships (function)
- Merge results using Reciprocal Rank Fusion (RRF) or weighted combination (function)

What gets retrieved depends on classification, query transformation, and the current subtask.

#### S4. Retrieval Evaluation
After retrieval, evaluate whether the results are actually useful before proceeding.

- Score each retrieved chunk for relevance (trait)
- Discard chunks below threshold (function)
- If all retrieved chunks score below threshold, the agent can:
  - Retry this step with a different query transformation
  - Fall back to alternative retrieval strategies (different data sources, web search)
  - Escalate or abstain ("I don't have enough information")

This prevents the "garbage in, garbage out" failure mode where bad context causes confident wrong answers.

#### S5. Reranking
Score each surviving chunk against the original query to determine actual usefulness.

- Score (query, chunk) pairs for relevance (trait) — sees both texts jointly, more accurate than embedding distance
- Filter down to top-k chunks (function)

Production systems often use two or three reranking stages chained together, each a different trait implementation (e.g. ColBERT → cross-encoder → LLM-based).

#### S6. Prompt Assembly
Build the prompt for this step's LLM call. All operations are (function):

- Assemble system instructions (agent role, personality, constraints, behavioral rules)
- Attach retrieved context from this step with source attribution markers
- Attach accumulated context from previous steps in this turn
- Attach conversation history (within session, possibly compacted)
- Attach tool results from previous steps
- Attach tool definitions
- Attach current plan state (if planning is active)
- Attach output format and grounding instructions

Strict separation of system instructions from user-supplied and tool-supplied content for prompt injection defense.

The prompt grows across steps as tool results and retrieved context accumulate. Working memory compaction may trigger if the context window fills.

#### S7. LLM Call
- Send assembled prompt to language model (trait)
- Track token usage and cost (function)

Model, temperature, and reasoning effort may vary by agent role and task type.

#### S8. Output Parsing
Extract structured parts from the LLM response. All operations are (function):

- Extract text response
- Extract tool call requests (function name, arguments)
- Extract structured data (decisions, classifications)
- Extract uncertainty signals
- Extract citations / source references

The parser determines what happens next:

- Tool calls → proceed to Agent Communication / Execution Gate / Tool Execution → new step
- Final text → proceed to Reflection
- Both → execute tools first, then evaluate

#### S9. Reflection / Self-Critique
The agent evaluates its own output before accepting it.

- Evaluate output quality against the question and retrieved context (trait)
- Determine outcome (function):
  - **Accept** — output is good, exit the inner loop
  - **Revise** — loop back to LLM Call with critique notes (same step, new LLM call)
  - **Retry** — needs fundamentally different context, trigger a new step
  - **Adjust plan** — the plan itself was wrong, revise it and continue

Reflection depth scales with task complexity and risk. Simple factoid answers get light reflection. High-stakes decisions get deep evaluation.

#### S10. Agent Communication
After reasoning and reflection, the agent may determine it needs another agent's involvement. This is not a tool call — the target is another autonomous reasoning entity.

- Route communication to target agent (function)
- Scope context for the receiving agent (function)

Four modes:

- **Delegation** — assign a subtask to another agent. The current inner loop pauses. The child agent runs its own full turn independently (with scoped context, not full parent history). When the child completes, its result feeds into the next step of the parent's inner loop.
- **Consultation** — ask another agent a question. The inner loop pauses until the answer arrives, then resumes in the current step with the answer added to context. The current agent stays in control.
- **Escalation** — hand the task up the hierarchy because it exceeds this agent's scope, authority, or capability. The current turn may end entirely.
- **Notification** — inform another agent without expecting a response. Fire and forget, the current step continues immediately.

Flow implications:

- Delegation and consultation pause the inner loop (like human-in-the-loop but with another agent)
- Escalation exits the inner loop and potentially ends the turn
- Notification does not interrupt the flow
- The receiving agent's response may change what tools are needed next, which is why this stage comes before Tool Execution

Information flow is filtered — the child/consulted agent only receives what it's authorized to see. Cross-agent trace IDs link the turns for observability.

#### S11. Execution Gate
Third security checkpoint. All operations are (function):

- Check tool authorization for this agent
- Validate arguments are within allowed bounds
- Check action target is within scope
- Classify tool risk tier (read-only / mutating / destructive)
- Route to human approval if required

Can reject the tool call, require confirmation, present a dry-run preview, or strip arguments.

#### S12. Tool Execution
If the LLM requested tool use and the execution gate passed. All operations are (function):

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

- Verify claims against retrieved sources (trait)
- Verify citations support the claims made (trait)
- Check policy compliance (function)
- Validate output schema (function)
- Trigger abstention if grounding confidence is too low (function)

If guardrails fail, the response can be sent back into the inner loop for revision (triggers a new step).

### E2. Response Assembly
Combine everything into a coherent response. All operations are (function):

- Assemble final text output
- Attach structured results (task updates, decisions)
- Attach citations with provenance
- Attach confidence indicators
- Attach step count and cost summary

### E3. Memory Consolidation
After the response is produced, update the agent's memory systems.

#### Memory Types

- **Working Memory** — the current context window. Managed by summarization and compaction when it fills. A compaction boundary preserves critical information while compressing older steps and turns.
- **Episodic Memory** — records of past interactions. "Last time this topic came up, we decided X." Stored as conversation summaries or extracted key-value facts. Retrieved by similarity to current context in future turns.
- **Semantic Memory** — the agent's accumulated knowledge base. The RAG corpus itself plus facts learned during interactions. Updated when new documents, specs, or decisions are produced.
- **Procedural Memory** — how to do things. Stored as tool schemas, workflows, learned procedures. Updated when new tools or processes are introduced.

#### Consolidation Actions

- Extract new facts from this turn (trait)
- Update or expire stale memories (function)
- Generate new embeddings for produced knowledge (function)
- Emit events to other agents or systems (function)
- Update task/project state (function)
- Write audit log entry for the complete turn (function)

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

### Approaches

- **Structure-aware chunking** (default) — parse documents using their inherent structure: headings, sections, paragraphs, lists, tables. Never split tables or code blocks.
- **Semantic chunking** — embed each sentence, find boundaries where similarity drops. Produces topically coherent chunks.
- **Hierarchical / parent-child** — store chunks at multiple granularities (sentence, paragraph, section). Retrieve at fine-grained level, expand to parent for context.
- **Proposition-based** — decompose into atomic factual statements. Most precise but expensive.

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

- Each child/consulted agent runs its own complete turn (outer frame + inner loop) independently
- The parent provides scoped context, not its full history
- Information flow between agents is filtered by the Retrieval Gate of the receiving agent
- Delegation follows the hierarchy: agents can delegate down or escalate up
- Cross-agent trace IDs link parent and child turns for end-to-end observability
- Delegation results feed into the parent's next step; escalation may end the parent's turn

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

- What embedding model strategy — shared across all agents, or domain-specific per agent role?
- How does the planning stage interact with multi-agent delegation? Does the planner assign subtasks to other agents?
- What is the compaction strategy for working memory across long-running sessions?
- How do we handle contradictions between episodic memory and current retrieved context?
- Should reflection depth be configurable per agent role or adaptive based on confidence?
- What is the right chunking granularity for different document types in our domain (specs, meeting transcripts, code)?
- What is the maximum step budget per turn? Should it vary by agent role or task classification?
- How are steps within a turn parallelized when the plan has independent subtasks?
