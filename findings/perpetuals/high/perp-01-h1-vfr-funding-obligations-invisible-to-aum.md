# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: High — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; component, model or conditional-impact proof
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# VFR Funding Obligations Invisible to AUM

## Description

With VFR funding owed by the pool, an LP redeems LP tokens before settlement → get_collective_position zeroes accrued funding fields → AUM omits the liability and overprices redemption.

Departing LP captures inflated redemption value; remaining LPs bear later funding outflow. Report records a pool-drain PoC.

The local harness creates funded LPs and long/short positions and configures VFR. Its assertions establish the intended positive funding and payout conditions, not an independently recomputed fair-redemption delta. The Surfpool setup deploys a local binary, migrates release/37 to release/38 and configures VFR; release/37 lacked VFR. Signature bypass/setup privileges are local scaffolding, so this is not proof of an exploit of the historical mainnet deployment.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Construct the aggregate position without funding fields

An LP exits while the custody owes funding to positions. get_collective_position explicitly populates accounting fields but defaults the funding fields, so the aggregate position omits those liabilities.

Source: `programs/[project]/src/state/custody.rs:783-799`

```rust
            Ok(Position {
                side: side.into(),
                price: if accounting.total_quantity.to_u128() > 0 {
                    math::checked_as_u64(
                        accounting.weighted_price.to_u128() / accounting.total_quantity.to_u128(),
                    )?
                } else {
                    0
                },
                size_usd: accounting.size_usd,
                borrow_size_usd: accounting.borrow_size_usd,
                unrealized_interest_usd,
                cumulative_interest_snapshot: accounting.cumulative_interest_snapshot,
                locked_amount: accounting.locked_amount,
                exit_fee_usd: accounting.exit_fee_usd,
                ..Position::default()
            })
```

#### Step 2 — Price pool assets from the incomplete aggregate

AUM feeds that aggregate into get_pnl_usd. LP redemption uses the resulting AUM before the funding recipient closes, leaving the later funding outflow with remaining LPs.

Source: `programs/[project]/src/state/pool.rs:1628-1640`

```rust
            if !custody.is_stable() {
                let long_collective_position =
                    custody.get_collective_position(Side::Long, current_time)?;

                // compute long aggregate unrealized pnl
                let long_pnl = self.get_pnl_usd(
                    &long_collective_position,
                    &token_trade_price,
                    &token_price.high(),
                    &custody,
                    current_time,
                    false,
                )?;
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/h1_vfr_aum_lp_socialization.rs`; function `poc_lp_redeems_at_inflated_aum`.

The local harness creates funded LPs and long/short positions and configures VFR. Its assertions establish the intended positive funding and payout conditions, not an independently recomputed fair-redemption delta. The Surfpool setup deploys a local binary, migrates release/37 to release/38 and configures VFR; release/37 lacked VFR. Signature bypass/setup privileges are local scaffolding, so this is not proof of an exploit of the historical mainnet deployment.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/h1_vfr_aum_lp_socialization.rs:880-927`

```rust
    // --- Assertions ---
    // The key assertion: the short received funding that was NOT reflected in AUM
    // at the time the opportunist LP redeemed. This means the opportunist got
    // an inflated share, and the remaining LP (lp_honest) absorbs the loss.
    let one_usd = utils::scale(1, Cortex::USD_DECIMALS);

    // 1. Funding obligation must be material
    assert!(
        funding_owed_to_short > 0,
        "CRITICAL: Short must have accrued funding (proves funding obligations exist)"
    );
    assert!(
        funding_owed_to_short > 100 * one_usd,
        "Funding obligation must be material (> $100). Got: ${}",
        funding_owed_to_short / one_usd
    );

    // 2. The LP redemption received more than it should have.
    //    Fair value per LP token should be (AUM - funding_obligation) / supply.
    //    Actual value was AUM / supply (funding invisible).
    //    The overpayment is the redeemer's pro-rata share of the invisible obligation.
    //
    //    Opportunist owns ~28% of the pool (added $500k to ~$1.25M existing).
    //    They redeemed half their tokens, so ~14% of pool.
    //    Overpayment ≈ funding_owed_to_short × 14%.
    //    We check a weaker bound: overpayment > 0, by verifying the LP got back
    //    more USDC than they would have if AUM had reflected the obligation.
    //
    //    Since the LP deposited $500k and the pool grew slightly from fees,
    //    half their tokens should return ~$250k. The excess above that share
    //    includes the invisible funding subsidy. We validate the short's funding
    //    payout came from pool assets the LP was priced against.
    assert!(
        short_payout > funding_owed_to_short / 2,
        "Short payout ({} USDC) must include a substantial portion of the \
         accrued funding (${} obligation). If it doesn't, funding wasn't \
         actually collected from the pool.",
        short_payout / one_usd as u64,
        funding_owed_to_short / one_usd,
    );

    // 3. The LP redeemed BEFORE the short closed, so the funding outflow hadn't
    //    hit the pool yet. The LP got full AUM pricing. Now verify the short's
    //    close actually extracted value from pool assets — proving the remaining
    //    LPs bear the cost the opportunist escaped.
    assert!(
        usdc_received > 0 && short_payout > 0,
        "Both LP redemption and short payout must be non-zero"
```
