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

# Fee Distribution: Off-Peg Stablecoin Creates Phantom Pool Value

## Description

Fee recipient times permissionless distribute_fees while USDC is above peg; USD debt becomes equal token quantity without oracle conversion, then debt is cleared at mismatched value.

Above-peg recipient excess bounded by fee debt and peg deviation; report PoC confirms $14.41 fee debt case. LP depeg-profit variants failed economics.

The final report records the above-peg recipient timing case on approximately 14.41 USD of fee debt. Profit is bounded by debt size and peg deviation, not TVL. The local test creates fee debt and changes oracle prices. Separate LP depeg/recovery variants were economically unviable; do not turn their modeled AUM mismatch into a demonstrated profitable drain.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Convert historical USD debt into the same raw token quantity

The model preserves the historical 1:1 settlement assumption. Fee recipients benefit above peg because equal token units are worth more USD than the debt extinguished.

Source: `programs/[project]/tests/[auditor]_tests/l8_distribute_fees_offpeg_aum.rs:105-122`

```rust
    // Accrue 1,000 USD of non-LP fees
    let non_lp_fee_usd: u64 = 1_000_000_000; // 1,000 USD at PRICE_DECIMALS=10
    pool.accrue_fee_debt(non_lp_fee_usd, 0, false).unwrap();

    // distribute_fees uses these values directly as token amounts:
    let lm_token_amount = pool.lm_fee_debt_usd;
    let protocol_token_amount = pool.protocol_fee_debt_usd;
    let total_token_transfer = lm_token_amount + protocol_token_amount;

    // The total token transfer equals the total USD debt — 1:1 assumption
    let total_usd_debt = pool.lm_fee_debt_usd + pool.protocol_fee_debt_usd;

    assert_eq!(
        total_token_transfer, total_usd_debt,
        "H-4 PoC: Token transfer amount must equal USD debt — the code \
         performs no price conversion. transfer={}, debt={}",
        total_token_transfer, total_usd_debt
    );
```

#### Step 2 — Permit above-peg prices through the one-sided gate

The gate only rejects a price below the lower threshold. It does not force a one-dollar conversion.

Source: `programs/[project]/src/instructions/public/position/distribute_fees.rs:250-263`

```rust
    // Check that the price of the redistribution mint which is a stable, is not lower than 0.99
    let fixe_percent_gate = 99 * 10u64.pow((Cortex::PRICE_DECIMALS - 2).into());

    let staking_reward_token_price = oracle.get_selected_price_by_feed_id_or_name(
        staking_reward_custody.oracle,
        staking_reward_custody.oracle_feed_id,
        &pool.multi_oracle_config,
        current_time,
    )?;

    require!(
        staking_reward_token_price.low().price > fixe_percent_gate,
        [project]Error::InvalidOraclePrice
    );
```

#### Step 3 — Identify the corrected conversion now in the snapshot

Current code converts each USD bucket through get_token_amount. This is the final reported repair and must not be quoted as the historical vulnerable calculation.

Source: `programs/[project]/src/instructions/public/position/distribute_fees.rs:268-275`

```rust
        let lm_stakers_fee_token_amount = staking_reward_token_price
            .get_token_amount(pool.lm_fee_debt_usd, staking_reward_custody.decimals)?;
        let protocol_fee_token_amount = staking_reward_token_price
            .get_token_amount(pool.protocol_fee_debt_usd, staking_reward_custody.decimals)?;
        let referrers_fee_token_amount = staking_reward_token_price
            .get_token_amount(pool.referrers_fee_debt_usd, staking_reward_custody.decimals)?;
        let manager_fee_token_amount = staking_reward_token_price
            .get_token_amount(pool.manager_fee_debt_usd, staking_reward_custody.decimals)?;
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/l8_distribute_fees_offpeg_aum.rs`; function `h4_integration_fee_recipient_timing_attack`.

The final report records the above-peg recipient timing case on approximately 14.41 USD of fee debt. Profit is bounded by debt size and peg deviation, not TVL. The local test creates fee debt and changes oracle prices. Separate LP depeg/recovery variants were economically unviable; do not turn their modeled AUM mismatch into a demonstrated profitable drain.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/l8_distribute_fees_offpeg_aum.rs:1641-1673`

```rust
    // distribute_fees sets token_amount = fee_debt_usd directly (no price conversion).
    // So recipients get `debt` tokens, each worth $1.003.
    //
    // NOTE: For non-autonom pools where pool.manager_fee_recipient == protocol_fee_recipient,
    // the protocol fee ATA receives BOTH protocol_fee_debt AND manager_fee_debt
    // (see distribute_fees.rs:320-329).
    assert_eq!(
        lm_tokens_received, lm_debt,
        "LM vault must receive exactly lm_fee_debt_usd tokens (1:1 bug). \
         got={}, debt={}",
        lm_tokens_received, lm_debt
    );

    // Protocol fee recipient receives protocol + manager fees in this pool config
    let expected_protocol_tokens = protocol_debt + manager_debt;
    assert_eq!(
        protocol_tokens_received, expected_protocol_tokens,
        "Protocol fee recipient must receive protocol_fee_debt + manager_fee_debt tokens (1:1 bug). \
         got={}, protocol_debt={}, manager_debt={}, sum={}",
        protocol_tokens_received, protocol_debt, manager_debt, expected_protocol_tokens
    );

    assert_eq!(
        referrer_tokens_received, referrer_debt,
        "Referrer vault must receive exactly referrers_fee_debt_usd tokens (1:1 bug). \
         got={}, debt={}",
        referrer_tokens_received, referrer_debt
    );

    // ── Step 8: Quantify overpayment ────────────────────────────────────────
    // Each token is worth $1.003, but the debt was denominated in $1.00.
    // Overpayment per recipient = tokens × ($1.003 - $1.00) = tokens × 0.003
    //
```
