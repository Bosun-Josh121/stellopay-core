## Automated Payment Scheduling Contract

This document describes the `payment_scheduler` smart contract added for issue `#204`.

### Scope

The payment scheduler contract automates recurring token transfers:

- cron-like recurring payment jobs
- centralized processing via `process_due_payments`
- retry handling for insufficient funds
- failure notifications via events
- pause / resume controls for employers

### Contract Location

- Contract: `onchain/contracts/payment_scheduler/src/lib.rs`
- Tests: `onchain/contracts/payment_scheduler/tests/test_scheduler.rs`

### Security Model

- `initialize` is **one-time only** and sets the contract owner (admin).
- Employers create jobs and optionally pre-fund the scheduler with tokens.
- **Anyone** can call `process_due_payments` to trigger scheduled executions.
- Only the **employer** that created a job can:
  - pause or resume the job
  - fund the job via `fund_job`
- Jobs never pull tokens directly from the employer; they only use funds held by the scheduler contract.

### Data Model

- `JobStatus`
  - `Active`, `Paused`, `Failed`, `Completed`
- `PaymentJob`
  - `id`, `employer`, `recipient`, `token`
  - `amount`, `interval_seconds`, `next_scheduled_time`
  - `max_executions`, `executions`
  - `max_retries`, `retry_count`
  - `status`

Storage keys:

- `Owner`: contract owner/admin
- `Initialized`: one-time init flag
- `NextJobId`: auto-incrementing job id
- `Job(id)`: stored `PaymentJob`

### Scheduling and Retry Semantics

- Jobs are considered **due** when
  - `status == Active`
  - current ledger timestamp `>= next_scheduled_time`
- For each due job, `process_due_payments`:
  - checks scheduler contract token balance
  - if balance `< amount`:
    - increments `retry_count`
    - if `retry_count > max_retries`, marks job `Failed`
    - otherwise pushes `next_scheduled_time` forward by `interval_seconds`
    - emits `job_failed` event
  - if balance `>= amount`:
    - transfers `amount` from scheduler contract to `recipient`
    - increments `executions`
    - resets `retry_count`
    - advances `next_scheduled_time` by `interval_seconds`
    - if `max_executions` is set and reached, marks job `Completed`
    - emits `job_executed` event

### Public API

- `initialize(owner)`
- `create_job(employer, recipient, token, amount, interval_seconds, start_time, max_executions, max_retries) -> id`
- `process_due_payments(max_jobs) -> processed`
- `pause_job(employer, job_id)`
- `resume_job(employer, job_id)`
- `fund_job(from, job_id, amount)`
- `get_job(job_id) -> Option<PaymentJob>`
- `get_owner() -> Option<Address>`

### Workflow Summary

1. Admin calls `initialize(owner)`.
2. Employer creates a recurring job via `create_job` and transfers tokens into the scheduler as needed using `fund_job` (or direct transfers).
3. An off-chain service or any actor periodically calls `process_due_payments(max_jobs)` to execute due jobs.
4. If the scheduler lacks sufficient funds for a job, it increments `retry_count` and emits a failure event.
5. Once `retry_count` exceeds `max_retries`, the job is marked as `Failed`.
6. Employers can `pause_job` / `resume_job` to control when payments run.

### Testing Focus

The test suite covers:

- initialization and owner invariants
- happy-path recurring execution for a fixed number of runs
- insufficient funds behavior, retries, and eventual success after top-up
- pause/resume flows and the absence of executions while paused

