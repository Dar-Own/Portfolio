# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Medium — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; component, model or conditional-impact proof
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Governance Inflation: Resolved Locked Stakes Recount on Sync

## Description

Staker finalizes locked stake, then any caller syncs voting power; sync tests amount>0 but omits is_resolved(), re-adding voting weight that finalize revoked.

A resolved stake can regain voting weight that finalization revoked because its recorded amount survives resolution; the report demonstrates inflated governance power.

The isolated test compares the historical calculation with a version excluding resolved stakes: 1,000,000 units at a 25,000 basis-point multiplier yield 2,500,000 phantom voting units versus zero. The separate integration file m1_integ_governance_voting_power_double_spend.rs:16-24 documents direct account manipulation to bypass a missing-account governance CPI problem. That fixture is not an ordinary on-chain lifecycle proof.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Finalize a matured stake while retaining its amount

Finalization marks the stake resolved after revoking governance power. The amount remains available for later token withdrawal.

Source: `programs/[project]/src/instructions/public/staking_p/finalize_locked_stake.rs:229-233`

```rust
    staking.nb_locked_tokens -= locked_stake.amount;

    locked_stake.resolved = true as u8;

    locked_stake.early_exit_fee = if params.early_exit {
```

#### Step 2 — Recount the resolved stake

A permissionless sync checks amount>0 but omits is_resolved. The same amount and multiplier therefore restore voting weight after finalization.

Source: `programs/[project]/src/instructions/public/sync_user_voting_power.rs:139-152`

```rust
    // Calculate voting power user should have
    let real_voting_power = {
        let mut real_voting_power = 0;

        // Staking
        {
            for locked_stake in user_staking.locked_stakes.iter() {
                if locked_stake.amount > 0 {
                    real_voting_power += math::checked_as_u64(
                        locked_stake.vote_multiplier as u128 * locked_stake.amount as u128
                            / Cortex::BPS_POWER,
                    )?;
                }
            }
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/m1_governance_resolved_stake_inflation.rs`; function `resolved_stake_still_counted_current_implementation`.

The isolated test compares the historical calculation with a version excluding resolved stakes: 1,000,000 units at a 25,000 basis-point multiplier yield 2,500,000 phantom voting units versus zero. The separate integration file m1_integ_governance_voting_power_double_spend.rs:16-24 documents direct account manipulation to bypass a missing-account governance CPI problem. That fixture is not an ordinary on-chain lifecycle proof.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/m1_governance_resolved_stake_inflation.rs:74-84`

```rust
    fn resolved_stake_still_counted_current_implementation() {
        let stakes = [make_stake(1_000_000, 25_000, true)];

        // Current (buggy) implementation counts it
        let buggy_power = calculate_voting_power(&stakes);
        assert_eq!(buggy_power, 2_500_000, "Bug: resolved stake is counted");

        // Fixed implementation does not
        let fixed_power = calculate_voting_power_fixed(&stakes);
        assert_eq!(fixed_power, 0, "Fixed: resolved stake is excluded");
    }
```
