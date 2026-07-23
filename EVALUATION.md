# Wanas Apps Backend Engineering Challenge: Evaluation Rubric

This document outlines the detailed grading and evaluation criteria used by the Wanas Apps engineering team to assess your submission.

## Evaluation Matrix

We evaluate your submission across five key pillars:

| Pillar | Focus Area | Exemplary (Score 5) | Acceptable (Score 3) | Poor (Score 1) |
| :--- | :--- | :--- | :--- | :--- |
| **Concurrency & Correctness** | Stock reservation under load | Zero overselling under high concurrency. Uses reliable database-level locking (optimistic/pessimistic) or transactional isolation. Proven via tests. | Overselling prevented, but relies on Node.js-level locks (single thread) or in-memory synchronization, which fails multi-instance scale. | Overselling occurs. Concurrency checks are bypassed under load. |
| **Architecture & OOP** | Separation of Concerns | Clean OOP design. Strict separation of routes, controllers, business services, and database layers. Strict type safety. | Mostly separated, but database operations leaking into controllers or state machine logic mixed with HTTP layer. | Monolithic controllers containing SQL/DB queries, state transitions, and business logic combined in one place. |
| **Resilience & Durability** | Background scheduler and state survival | Polling queue and SLA jobs survive server restarts by persisting state. Logging is structured, traceable, and detailed. | Background processes persist state, but log outputs are sparse/untraceable, or one order error crashes the entire poll batch. | Background tasks run purely in-memory and disappear on restart. Single-order errors crash the batch. |
| **Testing Rigor** | Automated Test Coverage | High-quality unit and integration tests covering the concurrency scenario, state machine rules, and webhook behaviors. | Tests exist and pass, but concurrency scenarios are not tested or mock setups are unstable. | No automated tests, or tests only verify trivial endpoints and ignore the state machine/concurrency. |
| **Documentation & DevDX** | Readme, choices, and startup ease | Setup runs flawlessly in a single command. `DECISIONS.md` provides clear architectural reasoning and trade-offs. | Setup works but requires manual configuration of SQLite files/nodes. `DECISIONS.md` is minimal. | Setup fails, or requires undocumented global dependencies. No `DECISIONS.md`. |
