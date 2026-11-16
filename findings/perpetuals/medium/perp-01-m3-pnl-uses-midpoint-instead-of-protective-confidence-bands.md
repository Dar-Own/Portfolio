# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Medium — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; mechanism assertions shown below
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# PnL Uses Midpoint Instead of Protective Confidence Bands

## Description

Trader closes or liquidates a position while oracle confidence is nonzero; get_pnl_usd uses midpoint for both sides instead of low for longs/high for shorts.

Exit payout is more favorable to trader and less conservative for pool than specified; confidence-dependent excess demonstrated in local artifacts.

The source test compares midpoint and protective calculations for the same constructed position and confidence band. Its positive-profit assertion is a unit-level counterexample, not a live payout receipt. Zero-confidence controls and separate short/loss cases exist in the same file.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Expose conservative confidence-adjusted prices

The oracle object provides lower and upper confidence bounds. A long exit would use the lower bound and a short exit the upper bound to protect the pool.

Source: `programs/[project]/src/state/oracle.rs:922-941`

```rust
    pub fn low(&self) -> Self {
        Self {
            price: self.price - self.confidence,
            exponent: self.exponent,
            confidence: 0,
            timestamp: self.timestamp,
            feed_id: self.feed_id,
            ..Default::default()
        }
    }

    pub fn high(&self) -> Self {
        Self {
            price: self.price + self.confidence,
            exponent: self.exponent,
            confidence: 0,
            timestamp: self.timestamp,
            feed_id: self.feed_id,
            ..Default::default()
        }
```

#### Step 2 — Use the midpoint for both directions

The PnL calculation instead selects token_trade_price.price for both sides. Nonzero confidence therefore yields a more favorable trader payout than the stated conservative policy.

Source: `programs/[project]/src/state/pool.rs:1392-1403`

```rust
        // Use High/Low price to protect the pool
        let exit_price = match Side::try_from(position.side)? {
            Side::Long => token_trade_price.price,
            Side::Short => token_trade_price.price,
            Side::None => return Err([project]Error::InvalidPositionState.into()),
        };

        let exit_fee_usd: u64 = if liquidation {
            position.liquidation_fee_usd
        } else {
            position.exit_fee_usd
        };
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/m3_pnl_midpoint_not_protective.rs`; function `m5_unit_long_midpoint_overstates_profit`.

The source test compares midpoint and protective calculations for the same constructed position and confidence band. Its positive-profit assertion is a unit-level counterexample, not a live payout receipt. Zero-confidence controls and separate short/loss cases exist in the same file.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/m3_pnl_midpoint_not_protective.rs:231-262`

```rust
    let pnl_midpoint = pool
        .get_pnl_usd(
            &position,
            &price_with_conf,
            &price_with_conf,
            &custody,
            1,
            false,
        )
        .unwrap();

    // PnL at low() ($24,500) — this is what the code SHOULD do for longs
    let pnl_protective = pool
        .get_pnl_usd(
            &position,
            &price_with_conf.low(),
            &price_with_conf,
            &custody,
            1,
            false,
        )
        .unwrap();

    // The midpoint profit must be strictly greater than the protective profit
    // because midpoint ($25,000) > low ($24,500) for a winning long
    assert!(
        pnl_midpoint.profit_usd > pnl_protective.profit_usd,
        "M-5 PoC: Long profit at midpoint ({}) must exceed profit at low() ({}).\n\
         The difference is the pool subsidy due to missing confidence bands.",
        pnl_midpoint.profit_usd,
        pnl_protective.profit_usd
    );
```
