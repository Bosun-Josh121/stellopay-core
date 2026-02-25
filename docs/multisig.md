## Multisig Contract for Critical Operations

This document describes the `multisig` smart contract added for issue `#202`.

### Scope

The multisig contract acts as a governance and safety layer in front of critical operations:

- contract upgrade approvals
- large outbound token payments from a shared wallet
- approvals for dispute resolution flows

The contract focuses on **threshold-based approvals**, clear **event logs** for off-chain automation, and a **break-glass emergency guardian**.

### Contract Location

- Contract: `onchain/contracts/multisig/src/lib.rs`
- Tests: `onchain/contracts/multisig/tests/test_multisig.rs`

### Security Model

- `initialize` is **one-time only** and must be called by the designated owner.
- A **fixed signer set** and **threshold** are stored on-chain.
- Only configured **signers** can:
  - propose new operations
  - approve existing operations
- Operations auto-execute once `approvals >= threshold`.
- An optional **emergency guardian** can execute any pending operation without satisfying the threshold (break-glass override).
- Large token payments are executed directly from the multisig contract balance using the Soroban token client.

### Data Model

Core types:

- `OperationKind`
  - `ContractUpgrade(Address, BytesN<32>)`
  - `LargePayment(Address, Address, i128)` as `(token, to, amount)`
  - `DisputeResolution(Address, u128, i128, i128)` as `(payroll_contract, agreement_id, pay_employee, refund_employer)`
- `OperationStatus`
  - `Pending`, `Executed`, `Cancelled`
- `Operation`
  - `id`, `kind`, `creator`, `status`, `created_at`, `executed_at`

Storage keys:

- `Owner`: configuration owner
- `Signers`: vector of signer addresses
- `Threshold`: required signatures count
- `EmergencyGuardian`: optional guardian address
- `OperationCounter`: auto-incrementing id
- `Operation(id)`: stored operation
- `Approvals(id)`: vector of signer addresses that approved

### Public API

- `initialize(owner, signers, threshold, emergency_guardian)`
- `propose_operation(proposer, kind) -> operation_id`
- `approve_operation(signer, operation_id)`
- `cancel_operation(caller, operation_id)`
- `emergency_execute(guardian, operation_id)`
- `get_operation(operation_id) -> Option<Operation>`
- `get_signers() -> Vec<Address>`
- `get_threshold() -> u32`
- `get_approvals(operation_id) -> Vec<Address>`

### Workflow Summary

1. Owner calls `initialize` with signer set, threshold, and optional guardian.
2. Any signer can call `propose_operation` to create a new operation (auto-approving as creator).
3. Additional signers call `approve_operation` until the approval count meets the threshold.
4. When `approvals >= threshold`, the contract:
   - executes `LargePayment` operations by transferring tokens from its balance
   - marks `ContractUpgrade` and `DisputeResolution` operations as executed for off-chain tooling to act on
5. Creator or owner can cancel a pending operation via `cancel_operation`.
6. The emergency guardian can call `emergency_execute` to force execution of a pending operation in break-glass scenarios.

### Testing Focus

The test suite covers:

- initialization invariants and threshold validation
- proposal creation, auto-approval by proposer, and approval tracking
- threshold-based execution for large token payments
- guardian emergency execution path
- cancellation rules (creator vs owner vs unauthorized)

### Security Notes

- Signer sets and thresholds should be chosen to balance **safety** (enough signatures required) and **operational usability**.
- Guardian usage should be rare and closely monitored; it bypasses normal threshold checks.
- For `ContractUpgrade` and `DisputeResolution` operations, off-chain services should:
  - subscribe to `operation_proposed`, `operation_approved`, and `operation_executed` events
  - verify that on-chain approvals meet policy before executing external actions (e.g., CLI upgrades, payroll dispute resolution).

