# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Low — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; component, model or conditional-impact proof
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# LP Sandwich: No Structural Hold Period

## Description

LP deposits just before liquidation/AUM increase then redeems immediately; no hold-time invariant prevents capturing a pro-rata event gain.

Structural sandwich opportunity; existing roughly 28bps round-trip fees/AUM pricing make own-capital baseline unprofitable, so material exploit needs changed economics or large event.

The local test uses funded attacker capital, constructed positions, timed price changes and liquidation. The final report describes roughly 28 basis-point round-trip cost and unprofitable baseline economics. It records a later pool-wide last-deposit-time guard, with a caveat that one depositor temporarily delays other withdrawals. No universal flash-loan profit or loss-free sandwich is claimed.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Redeem recently minted LP shares without a holding period

The historical characterization asserts that a deposit and redemption across three slots leave no LP balance. There is no per-holder minimum duration in that tested path.

Source: `programs/[project]/tests/[auditor]_tests/l11_lp_sandwich_characterization.rs:354-360`

```rust
    // Document: LP tokens have NO hold period — sandwich is technically possible
    // This assertion proves minting and redeeming in 3 slots works
    assert_eq!(
        utils::get_token_account_balance(&test_setup.program_test_ctx, attacker_lp_ata).await,
        0,
        "Attacker should have redeemed all LP tokens — no hold period enforced"
    );
```

#### Step 2 — Compare event capture against the actual round-trip cost

The attacker captures a pro-rata event change but pays deposit/withdrawal fees and enters at marked AUM. The fixture explicitly requires nonpositive profit; absence of a holding invariant is not sufficient for a profitable attack.

Source: `programs/[project]/tests/[auditor]_tests/l11_lp_sandwich_characterization.rs:374-388`

```rust
    // The attack becomes profitable only when a single liquidation event
    // transfers a large fraction of AUM from traders to the pool.

    // For this test scenario, the sandwich should NOT be profitable
    assert!(
        net_pnl <= 0,
        "Sandwich should not be profitable for a $75K position in a $23M pool"
    );

    // The round-trip cost should be in the range of 20-30 bps
    let cost_bps = round_trip_cost_f64 / deposit_amount_f64 * 10_000.0;
    assert!(
        cost_bps >= 15.0 && cost_bps <= 40.0,
        "Round-trip cost should be ~20 bps, got {:.1} bps", cost_bps
    );
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/l11_lp_sandwich_characterization.rs`; function `lp_sandwich_liquidation_characterization`.

The local test uses funded attacker capital, constructed positions, timed price changes and liquidation. The final report describes roughly 28 basis-point round-trip cost and unprofitable baseline economics. It records a later pool-wide last-deposit-time guard, with a caveat that one depositor temporarily delays other withdrawals. No universal flash-loan profit or loss-free sandwich is claimed.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/l11_lp_sandwich_characterization.rs:354-388`

```rust
    // Document: LP tokens have NO hold period — sandwich is technically possible
    // This assertion proves minting and redeeming in 3 slots works
    assert_eq!(
        utils::get_token_account_balance(&test_setup.program_test_ctx, attacker_lp_ata).await,
        0,
        "Attacker should have redeemed all LP tokens — no hold period enforced"
    );

    // Document: the round-trip tax is the primary sandwich defense.
    //
    // For a $1M deposit into a ~$23M pool (~4.3% share), profitability requires:
    //   liquidation_loss × (deposit / AUM) > round_trip_fees
    //   → liquidation_loss > round_trip_fees / (deposit / AUM)
    //   → liquidation_loss > ~$64K for this scenario
    //
    // With a 10 ETH × 5x long at $1,500 entry crashing to $1,200:
    //   - The position is underwater but the collateral absorbed is modest
    //   - The attacker's ~4% share of the absorbed amount is small
    //   - Round-trip fees (~28 bps) dominate
    //
    // The attack becomes profitable only when a single liquidation event
    // transfers a large fraction of AUM from traders to the pool.

    // For this test scenario, the sandwich should NOT be profitable
    assert!(
        net_pnl <= 0,
        "Sandwich should not be profitable for a $75K position in a $23M pool"
    );

    // The round-trip cost should be in the range of 20-30 bps
    let cost_bps = round_trip_cost_f64 / deposit_amount_f64 * 10_000.0;
    assert!(
        cost_bps >= 15.0 && cost_bps <= 40.0,
        "Round-trip cost should be ~20 bps, got {:.1} bps", cost_bps
    );
```
