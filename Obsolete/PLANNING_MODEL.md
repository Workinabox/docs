# Planning Model (Draft — for discussion)

**Status:** Draft. This captures the planning ontology we want to move toward. Nothing here is implemented yet, and the current backend `Work`/`Done` code is a throwaway placeholder that this model replaces (see [Relationship to the current code](#relationship-to-the-current-code)). Do not start building from this until we agree.

Source: a design conversation (raw transcript in `TEMP-PLANNING.md`). This document is the cleaned-up version of what was discussed.

---

## The core idea

The backend's current planning concept — `Work` — only expresses the *action*. It says nothing about *what* we are trying to achieve, *what must be true* for it to count as achieved, or *how* we verify it. Work is the action; it is not the reason for the action.

The conversation arrived at a small, **domain-neutral** ontology. It is deliberately not software-specific — the same vocabulary works for law, accounting, sales, HR, construction, operations. "Software company" is just one specialization of it. We do **not** bake Jira/Scrum/software vocabulary into the core.

---

## The base ontology

Seven concepts. **Four form a containment chain. Three — Check, Work, and Defect — cut across it.**

| Concept | One-line definition |
|---|---|
| **Project** | Container for coordinated change. |
| **Objective** | Desired future state. |
| **Requirement** | Mandatory condition that must be true for the objective. |
| **Specification** | Precise rule / description of required behavior. |
| **Check** | Verification procedure. Can attach to *any* of the other concepts. |
| **Work** | Executable, change-producing action. |
| **Defect** | Observed violation of what should be true. |

### The containment chain

```text
Project
  └─ Objective*
      └─ Requirement*
          └─ Specification*
```

Where `*` means "zero or more / many".

### Cross-cutting concepts

`Check`, `Work`, and `Defect` are **not** layers in the chain. They attach to or reference the things in it:

- A **Check** is a verification that can sit on *any* concept — a Project, an Objective, a Requirement, a Specification, a Work, or a Defect.
- A **Work** is the action taken to change reality; it references the chain (see below).
- A **Defect** is an observed violation of what should be true.

### Cardinality and satisfaction rules

- An **Objective has many Requirements.**
- An Objective is **satisfied iff all of its mandatory Requirements are satisfied.**
- A **Requirement has many Specifications.**
- Any concept may carry **zero or more Checks.**

### It is a graph, not a strict tree

Some requirements are shared across objectives. The canonical example:

> Requirement: *System must comply with GDPR* — applies to many objectives.

So internally these should be **linked objects, not only nested objects**. Conceptually it reads as a tree; structurally it must allow a Requirement (and likely a Specification) to be referenced by more than one parent.

---

## Work as a first-class object

Work must be **explicit**. It is *not* safely implied by the Objective.

> **Work = assigned change-producing action.** Not "task" in the casual sense.

The reasoning: a human developer can leave the "how" implicit because they hold the context. An **agent** cannot. If Work is implicit, agents may misunderstand the target, touch the wrong files, satisfy the objective but violate a constraint, skip verification, or claim completion without evidence. So Work — and, for agents, the Plan inside it — must be written down.

### What a Work carries

| Attribute | Meaning |
|---|---|
| **Purpose** | Why this work exists. |
| **Target** | What it acts on. |
| **Instruction** | What should be done. |
| **Output** | What artifact / change must result. |
| **Constraints** | What must not be violated. |
| **Checks** | How completion is verified. |

### How Work relates to the rest

```text
Work
  references Objective / Requirement / Specification / Defect
  produces   Change / Artifact / Result
  may contain or produce a Plan
  is verified by one or more Checks
```

Key boundary: **the Objective is the desired state; the Work is the action taken to reach it.** Work references the planning chain — it does not replace `Objective`, `Requirement`, or `Specification`.

---

## Plan vs. Work

Split "what" from "how":

- **Work** says *what action is required.*
- **Plan** says *how the agent intends to do it.*

```text
Work: Implement password reset endpoint.

Plan:
  1. Inspect existing auth routes.
  2. Add reset token table.
  3. Add endpoint.
  4. Add email dispatch.
  5. Add tests.
```

Humans usually leave the plan implicit. For agents we want it explicit.

---

## A worked example (end to end)

The same shape, shown outside software (opening an office) and inside it (password reset). The Check rows below are examples of Checks *attached to* a Specification — they could equally attach to a Work or a Requirement:

| Concept | Office example | Software example |
|---|---|---|
| **Project** | Open new office | — |
| **Objective** | Office ready for employees on August 1 | Users can reset forgotten passwords |
| **Requirement** | Office must be legally usable | System must not reveal whether an email exists |
| **Specification** | Fire inspection certificate approved before move-in | `POST /password-reset` always returns HTTP 202 with the same body |
| **Check** | Verify certificate exists and is approved | Call with existing + non-existing emails, verify identical responses |
| **Work** | Schedule fire inspection | Implement the password-reset request endpoint |
| **Defect** | Inspection failed — emergency lighting missing | (an observed violation of any of the above) |

---

## The software specialization

The core stays domain-neutral. A software company is one specialization:

```text
Software Project
Software Objective
Software Requirement
Software Specification
Software Check
Software Work
Software Defect
```

No Jira/Scrum vocabulary in the core.

---

## Relationship to the current code

The backend today has `Work` (flat aggregate: `id`, `project_id`, `title`, `description`, a list of `Done` items) and `Done` (an acceptance criterion). **Both are throwaway placeholders.** They will be changed or removed wholesale — this model does not attempt to preserve their current shape. `Project` is the only existing concept that carries straight over.

---

## Open questions

These were raised or left unresolved in the conversation. We should settle them before any modeling.

1. **Graph mechanics** — if Requirements/Specifications can be shared across Objectives, do we model them as standalone aggregates with links, or keep nesting and add cross-references? This affects aggregate boundaries (see `DDD.md`).
2. **Defect's place** — Defect references "what should be true" (Requirement/Specification) and spawns Work. Is Defect its own aggregate, and does it have its own lifecycle/states?
3. **Check's shape** — a Check can attach to any concept. Is it one polymorphic Check aggregate that references its target, or per-target checks? Does a Check have a result/state (pass/fail/pending)?
4. **Scope of the first step** — do we introduce the whole chain at once, or start with `Objective` (and maybe `Requirement`) and grow from there?

---

## Decided vs. undecided (summary)

**Decided in the conversation:**

- The seven-concept domain-neutral ontology and its definitions.
- The `Project → Objective → Requirement → Specification` containment chain.
- `Check` is **cross-cutting, not a hierarchy layer** — it can attach to any concept.
- Objective satisfied iff all mandatory Requirements satisfied; objectives have many requirements; requirements have many specifications.
- It is a graph, not a strict tree (shared requirements).
- `Work` stays first-class and explicit; it references the chain rather than replacing it.
- `Plan` (how) is split from `Work` (what), and made explicit for agents.
- Keep the core domain-neutral; software is a specialization.
- Today's `Work`/`Done` code is a throwaway placeholder, replaced by this model.

**Undecided:** everything in [Open questions](#open-questions).
