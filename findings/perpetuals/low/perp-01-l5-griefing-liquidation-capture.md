# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Low — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; component, model or conditional-impact proof
Remediation: Acknowledged — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Griefing: Liquidation Capture

## Description

Funded caller front-runs victim collateral/close operations with permissionless resolve_position_borrow_fees; shared pool write contention delays defensive transactions during adverse price movement.

Possible liquidation griefing and indirect LP gain; requires volatility, competing priority fees and near-liquidatable positions. Zero-interest early return limits no-work griefing.

The final report acknowledges this public maintenance function, notes zero-interest early return and transaction costs, and describes liquidation capture only under favorable market/ordering conditions. Test execution in a local runtime does not establish sustained real scheduler exclusion. Caller authority is ordinary; economic benefit additionally requires LP exposure.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Resolve another user position without owner authorization

Any funded signer can enter the fee-resolution instruction. The operation is intended as public maintenance but touches state also needed by defensive position actions.

Source: `programs/[project]/src/instructions/public/position/resolve_position_borrow_fees.rs:21-33`

```rust
#[derive(Accounts)]
pub struct ResolvePositionBorrowFees<'info> {
    /// #1
    #[account(mut)]
    pub signer: Signer<'info>,

    /// #2
    /// CHECK: Empty PDA, authority for token accounts
    #[account(
        seeds = [b"transfer_authority"],
        bump = cortex.load()?.transfer_authority_bump
    )]
    pub transfer_authority: AccountInfo<'info>,
```

#### Step 2 — Separate contention from interest-accounting theft

The control asserts unchanged borrow fees, loss and profit after crystallization. The reported issue is transaction contention during adverse price movement, not creation of additional debt by crystallization alone.

Source: `programs/[project]/tests/[auditor]_tests/l5_resolve_borrow_fees_permissionless.rs:453-470`

```rust
    assert_eq!(
        pnl_before.borrow_fee_usd, pnl_after.borrow_fee_usd,
        "L-4 PoC: borrow_fee_usd MUST be identical before and after crystallization \
         (accounting identity). Before: {}, After: {}",
        pnl_before.borrow_fee_usd, pnl_after.borrow_fee_usd
    );
    assert_eq!(
        pnl_before.loss_usd, pnl_after.loss_usd,
        "L-4 PoC: loss_usd MUST be identical before and after crystallization. \
         Before: {}, After: {}",
        pnl_before.loss_usd, pnl_after.loss_usd
    );
    assert_eq!(
        pnl_before.profit_usd, pnl_after.profit_usd,
        "L-4 PoC: profit_usd MUST be identical before and after crystallization. \
         Before: {}, After: {}",
        pnl_before.profit_usd, pnl_after.profit_usd
    );
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/l5_resolve_borrow_fees_permissionless.rs`; function `l4_integration_third_party_can_resolve_any_position`.

The final report acknowledges this public maintenance function, notes zero-interest early return and transaction costs, and describes liquidation capture only under favorable market/ordering conditions. Test execution in a local runtime does not establish sustained real scheduler exclusion. Caller authority is ordinary; economic benefit additionally requires LP exposure.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/l5_resolve_borrow_fees_permissionless.rs:394-430`

```rust
    let pnl_before = test_instructions::get_pnl(
        &test_setup.program_test_ctx,
        &test_setup.payer_keypair,
        &test_setup.pool_pda,
        &position_pda,
        Some(oracle_prices.clone()),
    )
    .await
    .unwrap();

    assert!(
        pnl_before.borrow_fee_usd > 0,
        "L-4 Setup: borrow fees must have accrued before the attack. Got: {}",
        pnl_before.borrow_fee_usd
    );

    // ── Alice (NOT the position owner) resolves Martin's borrow fees ──────────
    //
    // If an owner constraint existed this would return an error.
    // On the unpatched codebase it succeeds.
    let result = test_instructions::resolve_position_borrow_fees(
        &test_setup.program_test_ctx,
        alice, // ← third-party signer, not position owner (martin)
        &position_pda,
        ResolvePositionBorrowFeesParams {
            oracle_prices: Some(oracle_prices.clone()),
            multi_oracle_prices: None,
        },
    )
    .await;

    assert!(
        result.is_ok(),
        "L-4 PoC: resolve_position_borrow_fees MUST succeed when called by a \
         third party (alice) on martin's position. \
         If this is Err, an owner check was added (run the regression test instead). \
         Error: {:?}",
```
