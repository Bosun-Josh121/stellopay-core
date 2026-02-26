## Error Handling Guide

This guide summarizes the main error types and patterns used in the Stellopay payroll contracts, with a focus on **error codes**, **caller responsibilities**, and **recovery strategies**.

It is intentionally concise and is meant to complement the inline NatSpec-style comments and tests.

---

### Core Error Type: `PayrollError`

On-chain type: `PayrollError` (see `onchain/contracts/stello_pay_contract/src/storage.rs`).

This enum is annotated with `#[contracterror]` and `#[repr(u32)]`, so each variant has a stable numeric code surfaced to clients.

Selected variants and intended meaning:

- `DisputeAlreadyRaised (1)` – a dispute is already active for this agreement
- `NotInGracePeriod (2)` – operation requires the agreement to be in grace period
- `NotParty (3)` – caller is not employer/employee for this agreement
- `NotArbiter (4)` – caller is not the configured arbiter
- `InvalidPayout (5)` – `pay_employee + refund_employer` exceeds total locked funds
- `ActiveDispute (6)` – operation blocked while dispute is active
- `AgreementNotFound (7)` – referenced agreement id does not exist
- `NoDispute (8)` – attempting to resolve or query a non‑existent dispute
- `NoEmployee (9)` – employee index or address not present in agreement
- `NotActivated (10)` / `AgreementNotActivated (17)` – agreement must be active
- `Unauthorized (11)` – generic access control violation (e.g., wrong caller)
- `InvalidEmployeeIndex (12)` – out‑of‑range employee index
- `InvalidData (13)` – malformed or inconsistent stored data
- `TransferFailed (14)` – token transfer client call returned an error
- `InsufficientEscrowBalance (15)` – agreement escrow does not cover requested payment
- `NoPeriodsToClaim (16)` – time‑based escrow has no newly claimable periods
- `InvalidAgreementMode (18)` – operation incompatible with agreement mode
- `AgreementPaused (19)` – operation not allowed while agreement is `Paused`
- `AllPeriodsClaimed (20)` – all time‑based periods already claimed
- `ZeroAmountPerPeriod (21)` – invalid configuration, amount per period must be > 0
- `ZeroPeriodDuration (22)` – invalid configuration, duration per period must be > 0
- `ZeroNumPeriods (23)` – invalid configuration, number of periods must be > 0

These codes are surfaced in batch results:

- `PayrollClaimResult.error_code` (0 = success, otherwise `PayrollError` discriminant)
- `MilestoneClaimResult.error_code` (compact codes for milestone flows)

This allows off‑chain clients to distinguish **recoverable** issues (e.g., bad inputs, insufficient funds) from **hard** invariants (e.g., unauthorized caller).

---

### Error Handling Patterns

The contracts follow a small number of consistent patterns:

- **Typed errors for public functions**
  - Where practical, functions return `Result<_, PayrollError>` and use the enum above.
  - Clients should branch on `error_code` (for batches) or the `Result` error in direct calls.

- **`assert!`-style guards for internal invariants**
  - Many helper functions and validation checks use `assert!(...)` with a descriptive message.
  - These represent **programmer errors or violated assumptions** and are not intended as recoverable API contracts.

- **Access control failures**
  - Expressed as either `PayrollError::Unauthorized` or explicit `assert!(caller == expected, "...")`.
  - Recovery strategy: call from the proper address (e.g., employer, employee, arbiter) or adjust integration logic.

- **Mode and status checks**
  - Functions that depend on `AgreementMode` or `AgreementStatus` validate them first.
  - Typical responses:
    - `AgreementNotActivated`, `AgreementPaused`, `InvalidAgreementMode`
  - Recovery strategy: ensure agreement is created/activated and not paused before calling; for integrations, model the full lifecycle rather than calling arbitrarily.

---

### Recovery Strategies (By Category)

- **Authentication / Authorization**
  - Errors: `Unauthorized`, `NotParty`, `NotArbiter`
  - Recovery:
    - Re‑issue the transaction signed by the correct account (employer, employee, arbiter).
    - For UIs, hide actions that the current account cannot validly perform.

- **Configuration / Input Validation**
  - Errors: `ZeroAmountPerPeriod`, `ZeroPeriodDuration`, `ZeroNumPeriods`, `InvalidEmployeeIndex`
  - Recovery:
    - Validate inputs client‑side before submitting.
    - For batch operations, inspect `error_code` per item and surface field‑specific messages.

- **Lifecycle / Mode Mismatch**
  - Errors: `AgreementNotActivated`, `AgreementPaused`, `ActiveDispute`, `NotInGracePeriod`
  - Recovery:
    - Wait for the agreement to transition to the required state, or call the appropriate transition first (e.g., `activate_agreement`, `resume_agreement`, `finalize_grace_period`).
    - For disputes, ensure the dispute is raised or resolved in the correct order.

- **Funds and Transfers**
  - Errors: `InsufficientEscrowBalance`, `TransferFailed`, `InvalidPayout`
  - Recovery:
    - Fund the escrow or token balances appropriately before retrying.
    - Double‑check payout splits so that they do not exceed the total locked amount.

---

### Example: Handling Batch Payroll Claims

When calling `batch_claim_payroll`, the contract returns a `BatchPayrollResult`:

- check `total_claimed`, `successful_claims`, `failed_claims`
- iterate over `results: Vec<PayrollClaimResult>`
  - if `error_code == 0`, treat as success
  - otherwise, map the numeric code back to `PayrollError` for display or logging

This pattern generalizes to milestone batches (`BatchMilestoneResult`) and provides a **single transaction** with **per‑item diagnostics**, which is recommended for off‑chain orchestration and dashboards.  

