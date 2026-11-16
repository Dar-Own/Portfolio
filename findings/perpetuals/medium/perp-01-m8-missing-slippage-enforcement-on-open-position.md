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

# Missing Slippage Enforcement on Open Position

## Description

User submits open/increase with a price limit; market/oracle changes before execution, but handler never compares params.price with execution price and opens at oracle value.

User limit is ignored and collateral can be lost on adverse execution; report's v39 local Surfpool claim is not independently revalidated deployment evidence.

The test funds a trader, changes local oracle values and submits the stale user limit. The same file includes inverse regression expectations for a patched binary. The exploit and regression expectations apply to different source revisions.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Accept the user price limit as an instruction parameter

The parameter is documented as slippage protection. In the historical report revision, only nonzero input validation preceded assigning the oracle execution price.

Source: `programs/[project]/src/instructions/public/position/open_position_long.rs:158-165`

```rust
pub struct OpenPositionLongParams {
    pub price: u64, // requested. used for slippage protection
    pub collateral: u64,
    pub leverage: u32, // in BPS
    // Do not do that, except if you know the onchain price is fresh (i.e you did just update the price in a prior instruction or this is CPI)
    pub oracle_prices: Option<BatchPrices>,
    pub multi_oracle_prices: Option<MultiBatchPrices>,
}
```

#### Step 2 — Demonstrate the historical execution beyond the submitted limit

The historical exploit assertion expects a 1,500 limit to open at an updated 1,800 oracle price. These are historical test expectations, not behavior claimed for the mixed source snapshot.

Source: `programs/[project]/tests/[auditor]_tests/m8_missing_slippage_enforcement.rs:366-402`

```rust
    // ---------- 4. Assert: the position OPENED despite price breach ----------
    assert!(
        result.is_ok(),
        "M-2 PoC: open_position_long SHOULD succeed on unpatched code even \
         though oracle price ($1,800) exceeds the user's slippage limit ($1,500). \
         If this fails, the vulnerability may already be patched."
    );

    let (position_pda, _) = result.unwrap();

    // ---------- 5. Verify the position was opened at the WORSE price ----------
    let position = utils::get_zero_copy_account::<Position>(
        &test_setup.program_test_ctx,
        position_pda,
    )
    .await;

    eprintln!(
        "Step 2: Trader submitted price limit: ${}, actual entry: ${}",
        slippage_limit / 10u64.pow(Cortex::PRICE_DECIMALS as u32 - 2),
        position.price / 10u64.pow(Cortex::PRICE_DECIMALS as u32 - 2),
    );

    // The position opened at $1,800, NOT the user's $1,500 limit
    assert_eq!(
        position.price, updated_price,
        "Position entry price must equal the oracle price ($1,800), \
         proving params.price ($1,500) was completely ignored"
    );

    // Confirm the position price is WORSE than the user's limit
    assert!(
        position.price > slippage_limit,
        "Position price ({}) > slippage limit ({}) — user got a worse \
         entry than they authorized",
        position.price, slippage_limit
    );
```

#### Step 3 — Identify the now-present guard in the local snapshot

The inspected local source includes an explicit price comparison. The final response instead says the six lifecycle slippage guards were intentionally removed; the report body discusses a different revision. The conflict is retained rather than claiming this checkout proves the final deployment.

Source: `programs/[project]/src/instructions/public/position/open_position_long.rs:292-299`

```rust

    // Check slippage
    require_gte!(
        params.price,
        token_trade_price.price,
        [project]Error::MaxPriceSlippage
    );

```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/m8_missing_slippage_enforcement.rs`; function `m2_poc_open_position_ignores_slippage`.

The test funds a trader, changes local oracle values and submits the stale user limit. The same file includes inverse regression expectations for a patched binary. The exploit and regression expectations apply to different source revisions.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/m8_missing_slippage_enforcement.rs:349-372`

```rust
        &test_setup.program_test_ctx,
        trader,
        &test_setup.payer_keypair,
        &test_setup.pool_pda,
        &eth_mint,
        OpenPositionLongParams {
            price: slippage_limit, // User says: "max $1,500"
            collateral,
            leverage: 50_000, // 5x leverage
            oracle_prices: Some(
                utils::get_oracle_prices_as_chaos_labs_bundle(&test_setup.program_test_ctx).await,
            ),
            multi_oracle_prices: None,
        },
    )
    .await;

    // ---------- 4. Assert: the position OPENED despite price breach ----------
    assert!(
        result.is_ok(),
        "M-2 PoC: open_position_long SHOULD succeed on unpatched code even \
         though oracle price ($1,800) exceeds the user's slippage limit ($1,500). \
         If this fails, the vulnerability may already be patched."
    );
```
