# Roadmap

A walking-skeleton-first ordered list. Each item is the next thing worth doing. Architecture and devops dominate the early items so that everything that comes later (meetings, agent execution, product surface) has solid ground to land on. Existing work is slotted in at the point it makes sense to return to it.

This is a working list. Reorder freely.

When an item is done, overwrite it in place — do not remove it. The numbering and history of what was tackled in what order is part of the document's value.

1. Design and build the public-facing website of workinabox
2. Host the website
3. CI/CD for the website
4. Website reachable at workinabox.ai and workinabox.io
5. Workinabox can host git repos
6. Workinabox has a pipeline/actions system (GitHub Actions–style)
7. Design a Workinabox project management system (epics, stories, tasks, etc., Notion-style) before building it
8. Self-host workinabox and start using it to build workinabox
9. CI runs build, test, fmt, and clippy on every PR across all repos
10. One-command local dev bootstrap (backend + app + dev tooling)
11. Backend produces a versioned, reproducible container image in CI
12. Postgres persistence behind the existing repository traits
13. Database migration tooling and a migration test in CI
14. Structured logging with correlation IDs threaded through every crate
15. OpenTelemetry tracing wired from inbound request to outbound call
16. Single staging environment reachable from a public URL
17. Infrastructure-as-code for the staging environment
18. Secrets management (no secrets in repo, fetched at boot)
19. Identity provider chosen and integrated for human users
20. Authorization model: roles, scopes, and a middleware that enforces them
21. API versioning convention and a contract test harness
22. In-process domain event bus with a transactional outbox
23. Background job runner for async work
24. Error reporting and alerting on staging
25. Mobile app CI: signed builds distributed to TestFlight and Play internal
26. Documentation site published from `docs/`
27. Production environment promoted from staging, with backup/restore drilled
28. Feature flags
29. First end-to-end thin slice of an agent turn (Input → LLM → Response, no retrieval)
30. Task aggregate and board persistence
31. Retrieval pipeline: ingestion, chunking, vector store, hybrid search
32. Full inner-loop step with retrieval, tool execution, and reflection
33. Security gates (Input, Retrieval, Execution) with audit trail
34. Multi-agent communication (delegation, consultation, escalation, notification)
35. Memory consolidation across episodic, semantic, and procedural stores
36. Return to meetings: capture, transcription, and event extraction feeding the agent model
37. Voice-first UX in the mobile app on top of the meeting and agent stacks
