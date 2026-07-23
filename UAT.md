# Wanas Apps Backend Engineering Challenge: User Acceptance Testing (UAT) Manual

This manual provides a step-by-step User Acceptance Testing (UAT) script to verify that the Order Processing Automation Hub matches all functional requirements. 

You can use standard API clients (like Postman/Insomnia) or command-line tools like `curl` to execute these scenarios.

---

## 1. Initial State & Seed Verification
Before testing, make sure your SQLite database contains the necessary seeded records.

### Step 1.1: Verify Pre-seeded Customers
Verify that CRUD endpoints allow listing customers.
- **Action**: Send a `GET` request to `/customers`
- **Expected Response**: `200 OK` containing at least two pre-seeded customers.
  ```json
  [
    { "id": 1, "name": "Alice Smith", "email": "alice@example.com" },
    { "id": 2, "name": "Bob Jones", "email": "bob@example.com" }
  ]
  ```

### Step 1.2: Verify Pre-seeded Products & Stock
Verify that CRUD endpoints allow listing products and checking stock.
- **Action**: Send a `GET` request to `/products`
- **Expected Response**: `200 OK` containing at least three pre-seeded products with defined stock.
  ```json
  [
    { "id": 1, "name": "Super Fast Charger", "price": 29.99, "stock_on_hand": 10 },
    { "id": 2, "name": "Wireless Mouse", "price": 49.99, "stock_on_hand": 1 },
    { "id": 3, "name": "Ergonomic Keyboard", "price": 89.99, "stock_on_hand": 5 }
  ]
  ```

---

## 2. Sunny-Day Flow (Enforced Lifecycle)
Verify the standard order path: `pending` -> `paid` -> `processing` -> `shipped` -> `delivered`.

### Step 2.1: Create an Order
Create a new order for **Super Fast Charger** (Qty: 2, stock is 10).
- **Action**: Send a `POST` request to `/orders` with payload:
  ```json
  {
    "customer_id": 1,
    "line_items": [
      { "product_id": 1, "quantity": 2 }
    ]
  }
  ```
- **Expected Response**: `201 Created`. The order `status` must be `pending`, and `total` must calculate correctly ($59.98).
- **Audit Log Verification**: Check `status_history`. It must contain a record:
  `{ "status": "pending", "changed_at": "...", "reason": "Order created" }`.

### Step 2.2: Pay the Order (Auto-Transition to Processing)
Simulate inbound payment.
- **Action**: Send a `POST` request to `/webhooks/payment` with payload:
  ```json
  { "order_id": 1 }
  ```
- **Expected Response**: `200 OK`.
- **Status Check**: Retrieve the order (`GET /orders/1`). 
  - Status must be `processing` (since stock is available, it bypassed `paid` and moved straight to `processing`).
  - Available stock for Product 1 must now be **8** (`GET /products`).
- **Audit Log Verification**: Check `status_history`. It must contain two new entries:
  1. `{ "status": "paid", "changed_at": "...", "reason": "Payment received" }`
  2. `{ "status": "processing", "changed_at": "...", "reason": "Stock reserved successfully" }`

### Step 2.3: Idempotency Check
Call the payment webhook again.
- **Action**: Resend the same `POST` request to `/webhooks/payment` with payload `{ "order_id": 1 }`.
- **Expected Response**: `200 OK` (processed successfully, no duplicate audit entries or duplicate stock deductions).

### Step 2.4: Ship the Order
Manually ship the order via the dashboard control or API.
- **Action**: Send a `POST` request to `/orders/1/ship` (or use dashboard manual action).
- **Expected Response**: `200 OK`. The order `status` must now be `shipped`.
- **Audit Log Verification**: Check `status_history`. It must contain:
  `{ "status": "shipped", "changed_at": "...", "reason": "Shipped manually" }`.

### Step 2.5: Background Delivery Polling
Verify the background job successfully advances shipped orders to delivered.
- **Action**: Trigger a scheduler tick (e.g., wait for the next cron interval, or use the dashboard's "Force Scheduler Tick" control).
- **Expected Logs**: Standard output must print:
  `[Batch Started] Found 1 orders to poll...`
  If the mock carrier returns `delivered`, the console should print:
  `[Success] Order 1 advanced to delivered → [Batch Complete]`.
- **Expected Status**: Retrieve the order (`GET /orders/1`). The `status` must now be `delivered`.

---

## 3. Rainy-Day Flow & Guardrails (Invalid Transitions)
Verify that invalid operations are strictly blocked.

### Step 3.1: Jump States Bypassing Payment
Try to ship a `pending` order directly.
- **Action**: Create a new order `GET /orders/2` (status: `pending`). Send a `POST` request to `/orders/2/ship`.
- **Expected Response**: `400 Bad Request` or `422 Unprocessable Entity` stating that the transition from `pending` to `shipped` is invalid. The state must remain `pending`.

### Step 3.2: Modify Terminal States
Try to cancel a `delivered` order.
- **Action**: Send a `POST` request to `/orders/1/cancel` (Order 1 was already marked as `delivered`).
- **Expected Response**: `400 Bad Request` or `422 Unprocessable Entity` stating that terminal states cannot be modified.

---

## 4. Backorder & Restock Workflow
Verify that orders automatically transition to backordered when stock is insufficient, and recover when restocked.

### Step 4.1: Create Order Bypassing Stock
We have **1** unit of **Wireless Mouse** (Product 2). Create an order for **2** units.
- **Action**: Send a `POST` request to `/orders` with payload:
  ```json
  {
    "customer_id": 2,
    "line_items": [
      { "product_id": 2, "quantity": 2 }
    ]
  }
  ```
- **Expected Response**: `201 Created` with status `pending`.

### Step 4.2: Pay the Order (Auto-Transition to Backordered)
Simulate inbound payment.
- **Action**: Send a `POST` request to `/webhooks/payment` with the new order ID.
- **Expected Response**: `200 OK`.
- **Status Check**: Retrieve the order. 
  - Status must be `backordered`.
  - Product 2 stock must remain **1** (no stock is reserved since the request was backordered).
- **Audit Log Verification**: Check `status_history`. It must contain:
  1. `{ "status": "paid", "changed_at": "...", "reason": "Payment received" }`
  2. `{ "status": "backordered", "changed_at": "...", "reason": "Insufficient stock" }`

### Step 4.3: Restock Webhook Trigger
Simulate restocking Product 2.
- **Action**: Send a `POST` request to `/webhooks/restock` with payload:
  ```json
  {
    "product_id": 2,
    "quantity": 10
  }
  ```
- **Expected Response**: `200 OK`.
- **Status Check**: 
  - Product 2 stock must now be **9** (original 1 + 10 restocked - 2 reserved for the backorder).
  - Retrieve the backordered order. The status must now be `processing`.
- **Audit Log Verification**: Check `status_history`. It must contain:
  `{ "status": "processing", "changed_at": "...", "reason": "Stock allocated from restock" }`.

---

## 5. Concurrency & Overselling Safeguards
Verify that simultaneous orders for scarce items do not result in overselling.

### Setup Context
Product 3 (**Ergonomic Keyboard**) has exactly **1** unit in stock.

### Step 5.1: High-Concurrency Simulation
Simulate two different customers paying for Product 3 at the exact same moment.
- **Action**: 
  1. Create Order A: Customer 1 orders 1 unit of Product 3 (status: `pending`).
  2. Create Order B: Customer 2 orders 1 unit of Product 3 (status: `pending`).
  3. Send payment webhooks for both Order A and Order B **simultaneously** (e.g. using a script, or parallel `curl` commands in a terminal).
- **Expected Outcomes**:
  - **Exactly one** order (either A or B) must successfully transition to `processing`.
  - The other order must transition to `backordered`.
  - Available stock for Product 3 must be exactly **0** (`GET /products`). We must never see negative stock (`-1`).

---

## 6. SLA Monitoring & Escalation
Verify that stalled orders are automatically escalated.

### Setup Context
Choose a low default SLA threshold for testing (e.g., 5 seconds) or manipulate the database record.

### Step 6.1: Automatic SLA Trigger
Create a new order, pay for it, and let it sit in `processing`.
- **Action**: Wait past the SLA duration (or manually modify the order's `created_at` timestamp in the SQLite database to be in the past), then refresh the dashboard or query the SLA endpoint.
- **Expected Result**: 
  - The order must show `is_escalated: true` (or equivalent metadata indicator).
  - The dashboard must clearly highlight this order as **ESCALATED**.

---

## 7. Operational Dashboard & End-of-Day Digest
Verify operational visibility.

### Step 7.1: End-of-Day Digest
- **Action**: Request the End-of-Day (EOD) Digest via the dashboard or API endpoint.
- **Expected Result**: 
  - Total revenue recognized must sum only the orders that successfully transitioned to `paid`.
  - Count of escalated and backordered items must match the current system state accurately.
