# Quiz — AWS Step Functions

5 questions. Try them from memory first, then expand the answers.

---

### 1. You need to process 2 million IoT events per hour, each handled in under 10 seconds, and duplicates are acceptable. Standard or Express workflow — and why?

<details>
<summary>Answer</summary>

**Express.** It is built for high event rates and short durations (max 5 minutes), priced per request rather than per state transition, and offers at-least-once semantics — a fine trade-off when duplicates are acceptable. Standard would be far more expensive at 2M executions/hour and its exactly-once guarantee is unnecessary here.
</details>

---

### 2. A Lambda task fails with a transient SDK timeout. What is the difference between handling it with `Retry` vs `Catch`?

<details>
<summary>Answer</summary>

**Retry** re-executes the *same* state after a backoff — for transient errors you expect to succeed on a later attempt (SDK timeouts, throttling). **Catch** redirects to a *different* state when the error is not worth retrying or retries are exhausted — for fallback logic, compensation, or parking the payload in a DLQ. In `order-workflow.asl.json`, `Retry` handles `Lambda.ServiceException`; `Catch` routes to `RefundPayment` / `SendToDLQ`.
</details>

---

### 3. Why must every Lambda task in a workflow with retries be idempotent?

<details>
<summary>Answer</summary>

Because a retry re-runs the task's side effects. If `ProcessPayment` charges a card and then times out before responding, Step Functions retries it — without idempotency (e.g. an idempotency key on `orderId`), the customer gets charged twice. Design tasks so a repeated execution with the same input produces the same outcome: check-then-act on a unique key in DynamoDB, or pass idempotency keys to downstream APIs.
</details>

---

### 4. In the sample workflow, payment succeeds but inventory reservation fails. What pattern keeps the system consistent, and how is it implemented here?

<details>
<summary>Answer</summary>

The **saga pattern**: a distributed transaction broken into steps, each with a compensating action that undoes it. Here, each `Parallel` branch has its own `Catch` — a failed payment branch runs `RefundPayment`, a failed inventory branch runs `ReleaseInventory` — and the workflow-level `Catch` on `ChargeAndReserve` runs `CompensateOrder` for anything that already completed. No two-phase commit, no distributed locks; just explicit undo logic.
</details>

---

### 5. Your team already uses EventBridge to choreograph microservices with events. When would you still introduce Step Functions?

<details>
<summary>Answer</summary>

When the flow is **ordered, multi-step, and needs centralised control**: retries with backoff, human approval steps, timeouts, a single visual trace of one business transaction, and auditability of exactly what ran with what input. Choreography is great for decoupled fan-out ("order placed" → many independent consumers); orchestration wins when steps depend on each other and failure needs coordinated compensation. Many real architectures use both: Step Functions for the saga, EventBridge for the resulting domain events (see the SNS publish in the diagram).
</details>
