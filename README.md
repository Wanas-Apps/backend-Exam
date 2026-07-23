<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/white-logo.svg">
    <source media="(prefers-color-scheme: light)" srcset="assets/color-logo.svg">
    <img alt="Wanas Apps" src="assets/color-logo.svg" width="350">
  </picture>
</p>

# Backend Engineering Challenge: Order Processing Automation Hub

Welcome to the Wanas Apps engineering challenge! We respect your time, so we have designed this task to be completed in roughly 2 to 3 days. Treat this timeframe as a ceiling — a smaller scope executed with high quality is always better than a large scope stretched thin.

## 1. Background & Problem

Our e-commerce order handling currently relies on a mix of manual dashboard steps, spreadsheets, and one-off scripts. As our order volume grows, this setup is breaking down in two critical ways:
- **Stalled Orders**: Orders get stuck between stages (e.g., paid but never fulfilled) and nobody notices until a customer complains.
- **Concurrency & Overselling**: When multiple orders come in simultaneously for a low-stock product, we oversell because stock is checked manually without reliable concurrency guards.

Your mission: Build a backend system that owns the order lifecycle end-to-end, automates transitions where possible, prevents overselling under concurrent load, and automatically surfaces stuck orders.

## 2. Core Goals

- **Enforced Lifecycle**: Every order moves through a strict state machine; invalid state jumps are impossible.
- **Concurrency Safety**: Stock is reserved correctly during simultaneous orders — we never oversell.
- **Proactive SLA Monitoring**: Orders that stall past their expected time-to-ship are flagged automatically.
- **Durability & Observability**: Automatic behaviors survive a process restart, and background processes are traceable through clear logging.
- **Operational Visibility**: A dashboard allows our ops team to see what is in progress, stuck, backordered, and completed at a glance.

## 3. Required Technology Stack

To minimize local setup friction, candidates **must** use the following technology stack for this challenge:
- **Backend Framework**: Node.js (e.g., Express, NestJS)
- **Database**: SQLite

Using these technologies is required. If you use any specific database client or ORM (e.g., Sequelize, TypeORM, Prisma, or raw SQL), please document your choice and configuration in your `DECISIONS.md`.

## 4. Out of Scope (What NOT to build)

- Customer-facing storefront or checkout UI.
- Authentication/authorization (assume a single ops user).
- Real payment or carrier integrations (use the simulated mocks detailed below).
- Deployment or hosting (local execution only).

---

## Technical & Functional Requirements

### 5. REST API

Build CRUD endpoints for Customer, Product, and Order resources.
- **Order fields**: `id`, `customer_id`, `line_items` (product, qty, price), `total`, `status`, `created_at`, `status_history`.
- **Product fields**: `id`, `name`, `price`, `stock_on_hand`.
- **Customer fields**: `id`, `name`, `email`.

> [!TIP]
> **SQLite Modeling Hint**: Since SQLite is relational, you may model `line_items` and `status_history` using separate relational tables (e.g., `order_items` and `order_status_history` tables) or leverage SQLite's native `JSON` support. Document your design choice in `DECISIONS.md`.

### 6. Workflow Engine (State Machine)

Orders must strictly follow the defined lifecycle. Invalid transitions (e.g., pending directly to shipped) must be rejected with a clear `4xx` error and never silently fail.
Every transition must append an entry to the `status_history` array acting as an audit trail.

| From State | To State | Trigger / Condition |
| :--- | :--- | :--- |
| `pending` | `paid` | Inbound payment webhook |
| `paid` | `processing` | Automatic, conditional on successful stock reservation |
| `paid` | `backordered` | Automatic, when required stock is unavailable |
| `backordered` | `processing` | Restock webhook or scheduler makes stock available |
| `processing` | `shipped` | Manual action via dashboard |
| `shipped` | `delivered` | Scheduled job (polls mock carrier tracking status) |
| `*` (Any Active) | `cancelled` | Manual action from any non-terminal state |

> [!IMPORTANT]
> **Terminal States**: `delivered` and `cancelled` are terminal states. Once an order enters these states, no further state transitions are permitted.

### 7. Concurrency & Stock Reservation

Reserving stock must decrement available inventory for the ordered products safely.
If two orders for the same scarce product are paid at effectively the exact same time, and only one can be fulfilled, exactly one must reserve the stock. The other must seamlessly transition to `backordered`. Correctness must not rely on requests being processed serially.

**Crucial**: You must explicitly document your database-level concurrency strategy (e.g., optimistic versioning, pessimistic locking) in `DECISIONS.md`.

### 8. Background Jobs, Resilience & Observability

Implement a durable scheduled job (using cron, a task scheduler, or an in-process timer) to handle ongoing tasks.
- **Delivery Polling**: Polls the mock carrier endpoint for shipped orders and advances them to delivered.
- **Durability**: Any pending background task must survive a server restart. It cannot rely purely on volatile in-memory state.
- **Resilience**: If one order's polling fails, the batch must continue processing the remaining orders.
- **Logging**: Background operations must not be a black box. The system must emit structured, traceable standard output logs (e.g., `[Batch Started] Found 5 orders to poll → [Success] Order 123 advanced to delivered → [Batch Complete]`).

### 9. SLA Monitoring & Escalation

Each order has a target time-to-ship (choose and document reasonable default durations). If an order sits in `paid`, `processing`, or `backordered` past its deadline, it must be automatically flagged as escalated and surfaced on the dashboard. This monitoring must survive server restarts.

> [!NOTE]
> **Escalation Design**: Escalation should be represented as a boolean flag or a metadata attribute (e.g., `is_escalated` or `escalated_at`), **not** as a state in the core workflow state machine. This allows the order to remain in its current state (such as `paid` or `backordered`) while being highlighted operationally.

### 10. Webhooks & Mocks

- **Payment Webhook** (`POST /webhooks/payment`): Simulates a payment provider. Must be idempotent (calling it twice for the same order does not duplicate the audit log). Referencing a non-existent order returns `404`.
  - **Request Body**: `{ "order_id": "string_or_number" }`
  - **Response**: `200 OK` with `{ "success": true, "message": "..." }`
- **Restock Webhook** (`POST /webhooks/restock`): Reports replenished stock and moves eligible backordered orders to processing.
  - **Request Body**: `{ "product_id": "string_or_number", "quantity": 10 }`
  - **Response**: `200 OK` with `{ "success": true, "message": "..." }`
- **Mock Carrier** (`GET /mock/carrier/:orderId/tracking`): Lives inside your application to stand in for a third-party shipping API.
  - **Response**: `200 OK` with `{ "order_id": "...", "status": "in_transit" | "delivered" }`

### 11. Dashboard & End-of-Day Digest

Provide an interface that allows an ops user to:
- View all orders and their current status, highlighting escalated and backordered items.
- View an order's audit log.
- Execute manual actions (Ship, Cancel).
- Trigger demo controls (fire payment webhook, restock webhook, force scheduler tick).
- View an End-of-Day Digest summarizing: orders fulfilled, orders escalated, orders backordered, and total revenue recognized.

> [!NOTE]
> - **Revenue Recognition**: For the purposes of the End-of-Day Digest, revenue is considered "recognized" once the order transitions to the `paid` state.
> - **Dashboard Simplicity**: The dashboard does not need to be a complex frontend framework. A simple server-rendered HTML page, or even a well-structured Postman workspace acting as the "dashboard" to trigger actions and view the digest, is perfectly acceptable. Keep your focus on the backend logic.

---

## Deliverables & Evaluation

Please submit your work by creating a **public GitHub repository** and sending us the link once you are finished.
- **Code & Commit History**: Full implementation with incremental commit history (do not squash into a single commit).
- **README.md**: Clear setup and run instructions. Someone who has never seen the code should be able to spin it up using one or two commands.
- **UAT Verification**: All API endpoints and behavior flows must be verified against the [User Acceptance Testing (UAT) Manual](UAT.md).
- **Automated Tests**: Minimum coverage must include valid/invalid state transitions and the concurrency/no-oversell behavior.
- **DECISIONS.md**: A short document detailing your architecture choices, your database-level concurrency strategy, and what you would change given more time/scale.
- **Live Walkthrough**: Be prepared for a 15–20 minute session where you will run the app and demonstrate the workflow paths, edge cases, and manual overrides.

### Evaluation Rubric

Submissions are graded objectively across five key pillars: Concurrency & Correctness, Architecture & OOP, Resilience & Durability, Testing Rigor, and Documentation & DevDX.

For a detailed breakdown of the grading criteria and scoring rubric, see the [Evaluation Rubric (EVALUATION.md)](EVALUATION.md).

### Optional Extensions

If you finish the core scope early, you may pick at most one or two of the following to implement.
- **Enterprise ERP/CRM Outbound Sync**: Emit a clean, structured JSON payload whenever an order hits the `delivered` state, simulating an outbound webhook push to an external enterprise system.
- **Reports & Analytics**: Trends on average time-in-state, backorder rates, or top delaying products.
- **Configurable Workflows**: Make the state machine data-driven (rules defined in config, not hardcoded).
- **AI/Agent Features**: An LLM that generates the End-of-Day digest as a prose summary, or a support chatbot that answers "where is my order?" based on live state.
