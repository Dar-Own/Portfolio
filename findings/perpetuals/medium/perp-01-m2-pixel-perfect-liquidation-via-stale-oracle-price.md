# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Medium — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; mechanism assertions shown below
Remediation: Acknowledged — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Pixel-Perfect Liquidation via Stale Oracle Price

## Description

An actor retains a signed flash-crash batch and submits it within the default seven-second use-age limit, before a newer accepted timestamp prevents replay → although the market has recovered, the accepted old price satisfies liquidation.

Recovered position can be liquidated at attested stale price; liquidation fees flow to pool, so attacker gain requires LP exposure. Team considers threshold crossing valid liquidation.

Tests create funded positions, set oracle prices and warp local time. The final report acknowledges the behavior as valid attested-price liquidation semantics, with monotonic timestamp protection after newer updates. Liquidation fees flow to the pool; an attacker needs LP exposure and favorable economics to benefit. Do not present this as signature forgery or an unlimited replay window.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Retain a signed crash price within the usage window

A valid price may remain usable for the configured freshness period, reported as seven seconds by default. The 300-second ingestion bound does not create a 300-second liquidation window.

Source: `programs/[project]/src/state/oracle.rs:1068-1078`

```rust
    pub fn check_price_staleness_with(
        &self,
        current_time: i64,
        staleness_seconds: i64,
    ) -> Result<()> {
        require!(
            self.timestamp + staleness_seconds >= current_time,
            [project]Error::StaleOraclePrice
        );
        Ok(())
    }
```

#### Step 2 — Compare the recovered-price control

The control submits the recovered price and expects liquidation to fail. The integration variants then examine execution with the retained crash quote before the usage window closes and before a newer accepted timestamp blocks replay.

Source: `programs/[project]/tests/[auditor]_tests/m2_pixel_perfect_liquidation_stale_price.rs:299-319`

```rust
        .await;

    // Sanity: with the fresh $1,500 price, liquidation fails
    let recovered_prices =
        utils::get_oracle_prices_as_chaos_labs_bundle(&test_setup.program_test_ctx).await;

    let liq_at_recovered = test_instructions::liquidate_long(
        &test_setup.program_test_ctx,
        alice,
        &test_setup.payer_keypair,
        &test_setup.pool_pda,
        eth_mint,
        &position_pda,
        Some(recovered_prices),
    )
    .await;

    assert!(
        liq_at_recovered.is_err(),
        "M-4 Sanity: position must NOT be liquidatable at recovered $1,500."
    );
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/m2_pixel_perfect_liquidation_stale_price.rs`; function `m4_integration_stale_batch_during_oracle_downtime`.

Tests create funded positions, set oracle prices and warp local time. The final report acknowledges the behavior as valid attested-price liquidation semantics, with monotonic timestamp protection after newer updates. Liquidation fees flow to the pool; an attacker needs LP exposure and favorable economics to benefit. Do not present this as signature forgery or an unlimited replay window.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/m2_pixel_perfect_liquidation_stale_price.rs:441-477`

```rust
    //   - check_price_staleness_with: T=0 + 7 >= T=5 -- passes (within 7s window)
    //
    // The liquidation uses the stored $1,300 price -> position is liquidatable.
    let liq_result = test_instructions::liquidate_long(
        &test_setup.program_test_ctx,
        alice,
        &test_setup.payer_keypair,
        &test_setup.pool_pda,
        eth_mint,
        &position_pda,
        Some(crash_batch.clone()),
    )
    .await;

    assert!(
        liq_result.is_ok(),
        "M-4 PoC: Liquidation with 5-second-old stale batch MUST succeed \
         on unpatched code (within staleness_seconds=7 window). \
         Error: {:?}",
        liq_result.err()
    );

    // Verify position is closed
    let position_gone = {
        let mut ctx = test_setup.program_test_ctx.write().await;
        ctx.banks_client
            .get_account(position_pda)
            .await
            .unwrap()
            .is_none()
    };

    assert!(
        position_gone,
        "M-4 PoC: Position PDA must be closed after liquidation."
    );

```
