## Token Vesting Contract

This document describes the `token_vesting` smart contract added for issue `#198`.

### Scope

The token vesting contract manages time-based release of tokens for employees and other beneficiaries:

- linear vesting over a time range
- single-time cliff vesting
- custom step schedules
- early release with admin approval
- revocation of unvested tokens for terminated employees

### Contract Location

- Contract: `onchain/contracts/token_vesting/src/lib.rs`
- Tests: `onchain/contracts/token_vesting/tests/test_vesting.rs`

### Security Model

- `initialize` is **one-time only** and sets the contract owner (admin).
- Employers must **escrow the full vesting amount up front** at schedule creation.
- Only the **beneficiary** can claim vested tokens for their schedule.
- Only the **contract owner** can approve early release of unvested tokens.
- Only the **employer** that created a revocable schedule can revoke it.
- Revocation refunds only the **unvested** portion; vested amounts remain claimable by the beneficiary.

### Data Model

Core types:

- `VestingKind`
  - `Linear`
  - `Cliff`
  - `Custom`
- `VestingStatus`
  - `Active`, `Revoked`, `Completed`
- `CustomCheckpoint`
  - `time`: absolute timestamp
  - `cumulative_amount`: total vested amount at `time`
- `VestingSchedule`
  - `id`, `employer`, `beneficiary`, `token`
  - `kind`, `status`, `revocable`, `revoked_at`
  - `total_amount`, `released_amount`
  - `start_time`, `end_time`, optional `cliff_time`
  - `checkpoints`: used for `Custom` schedules

Storage keys:

- `Owner`: contract owner/admin
- `Initialized`: one-time initialization flag
- `NextScheduleId`: auto-incrementing schedule id
- `Schedule(id)`: stored `VestingSchedule`

### Vesting Logic

- **Linear**
  - Vesting grows proportionally between `start_time` and `end_time`.
  - Vested amount = `total * (now - start) / (end - start)`, capped at `total`.
- **Cliff**
  - No vesting before `cliff_time`.
  - 100% vests at `cliff_time`.
- **Custom**
  - Uses ordered `CustomCheckpoint` entries.
  - Vested amount = last `cumulative_amount` with `time <= now`, capped at `total`.
- When a schedule is **revoked**, `revoked_at` freezes further vesting; vested amount at that time remains claimable.

### Public API

- `initialize(owner)`
- `create_linear_schedule(employer, beneficiary, token, total_amount, start_time, end_time, cliff_time, revocable) -> id`
- `create_cliff_schedule(employer, beneficiary, token, total_amount, cliff_time, revocable) -> id`
- `create_custom_schedule(employer, beneficiary, token, total_amount, checkpoints, revocable) -> id`
- `claim(beneficiary, schedule_id) -> amount`
- `approve_early_release(admin, schedule_id, amount) -> released`
- `revoke(employer, schedule_id) -> refunded_amount`
- `get_schedule(id) -> Option<VestingSchedule>`
- `get_vested_amount(id) -> i128`
- `get_releasable_amount(id) -> i128`
- `get_owner() -> Option<Address>`

### Workflow Summary

1. Admin calls `initialize(owner)`.
2. Employer funds and creates a vesting schedule (linear, cliff, or custom).
3. Beneficiary monitors `get_vested_amount` / `get_releasable_amount` and calls `claim` to pull vested tokens.
4. Admin can use `approve_early_release` to unlock part of the **unvested** portion ahead of schedule.
5. Employer can call `revoke` on revocable schedules to reclaim unvested tokens when an employee is terminated; beneficiary can still claim any vested remainder.

### Testing Focus

The test suite covers:

- initialization and owner behavior
- linear vesting progression and full completion
- cliff vesting with revocation before vesting
- custom checkpoint schedules
- early release semantics and admin-only access
- revocation behavior and refund calculation

