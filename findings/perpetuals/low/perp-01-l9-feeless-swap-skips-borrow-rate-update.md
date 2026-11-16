# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Low — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; mechanism assertions shown below
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Feeless Swap Skips Borrow Rate Update

## Description

Ordinary open/increase-with-swap triggers internal feeless swap that changes custody.assets.owned, then early returns before updating receiving custody's borrow rate.

Old higher utilization/rate overcharges borrowers until next touch; report estimates attacker costs exceed excess-interest gain in tested composition.

The source model compares stale and immediately refreshed rates and asserts excess interest. The final report calls the interaction with L1 additive, not multiplicative, and estimates repeated attack costs exceed excess gain. This is a stale-rate accounting defect with bounded economic feasibility, not a proven profitable cycling strategy.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Change custody liquidity and return on the feeless path

An internal swap adds/removes owned assets, then returns before the rate refresh. Open/increase-with-swap can invoke this path through the protocol transfer authority.

Source: `programs/[project]/src/instructions/public/liquidity/swap.rs:324-345`

```rust
        // Account for all fees there as part of the custody. distribute_fees will take care of removing it from the custody
        receiving_custody.assets.owned += deposit_amount;

        dispensing_custody.collected_fees.swap_usd = dispensing_custody
            .collected_fees
            .swap_usd
            .wrapping_add(fees_out_usd);

        dispensing_custody.volume_stats.swap_usd =
            dispensing_custody.volume_stats.swap_usd.wrapping_add(
                dispensed_token_price_low
                    .get_asset_amount_usd(amount_out, dispensing_custody.decimals)?,
            );

        dispensing_custody.assets.owned -= withdrawal_amount;
    }

    // Swap the collected fee_amount to stable and send to staking rewards
    // when it's an internal swap, no fees swap is done
    if feeless_swap {
        return Ok(());
    }
```

#### Step 2 — Skip the refresh positioned after that return

Both custody rate updates are below the early exit. The receiving custody can retain its old higher utilization-based rate until another operation updates it.

Source: `programs/[project]/src/instructions/public/liquidity/swap.rs:372-377`

```rust

    // Update borrow rate (because we added/removed tokens as "owned" in the custodies)
    {
        receiving_custody.update_borrow_rate(current_time)?;
        dispensing_custody.update_borrow_rate(current_time)?;
    }
```

#### Step 3 — Charge elapsed time using the stale rate

Subsequent interest accumulation multiplies the stale rate by elapsed time. Correcting the rate later does not retroactively erase that interval.

Source: `programs/[project]/src/state/custody.rs:471-479`

```rust
    pub fn get_cumulative_interest(&self, current_time: i64) -> Result<u128> {
        if current_time > self.borrow_rate_state.last_update {
            let cumulative_interest = math::checked_ceil_div(
                (current_time - self.borrow_rate_state.last_update) as u128
                    * self.borrow_rate_state.current_rate as u128,
                3_600,
            )?;

            Ok(self.borrow_rate_state.cumulative_interest.to_u128() + cumulative_interest)
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/fuzzing_suite/fuzz_swap_stale_rate_interaction.rs`; function `c1_stale_rate_overcharges_borrowers`.

The source model compares stale and immediately refreshed rates and asserts excess interest. The final report calls the interaction with L1 additive, not multiplicative, and estimates repeated attack costs exceed excess gain. This is a stale-rate accounting defect with bounded economic feasibility, not a proven profitable cycling strategy.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/fuzzing_suite/fuzz_swap_stale_rate_interaction.rs:140-160`

```rust
fn c1_stale_rate_overcharges_borrowers() {
    let max_rate = 100_000u64; // 0.01%/hr

    // Initial: 8M locked, 10M owned → utilization 80% → high rate
    let initial_owned = scale(10_000_000, 6);
    let locked = scale(8_000_000, 6);

    // Swap adds 5M → owned becomes 15M → utilization 53% → lower rate
    let owned_after_swap = scale(15_000_000, 6);

    let stale_window = 3600i64; // 1 hour stale

    let rate_before = linear_rate(initial_owned, locked, max_rate);
    let rate_correct = linear_rate(owned_after_swap, locked, max_rate);

    // Interest with stale rate (old high rate for 1 hour)
    let (total_stale, _stale_portion, _) =
        simulate_stale_then_correct(initial_owned, locked, owned_after_swap, stale_window, 0, max_rate);

    // Interest with correct rate (immediately updated)
    let total_correct = simulate_correct_rate(owned_after_swap, locked, stale_window, max_rate);
```
