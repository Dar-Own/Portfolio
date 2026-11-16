# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Low — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; component, model or conditional-impact proof
Remediation: Resolved (published label; residual caveat) — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Griefing: Pool-Wide Write-Lock

## Description

Any funded wallet repeatedly invokes permissionless distribute_fees, including no-work calls; mutable pool/staking/oracle accounts create lock contention without caller authorization.

Pool-wide position/staking delay described; profitable liquidation composition remains conditional. Final label Resolved coexists with fee-present residual locking.

The final report labels this Resolved through a zero-debt early return but also retains permissionless fee-present contention and older no-op locking prose. An early return does not change Solana message account writability. The available test does not settle the scheduler-level question. The integration tests model price movement and delays; indirect LP gain is conditional.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Invoke fee distribution from an arbitrary funded signer

The caller field has no admin/keeper restriction. Account metadata includes shared mutable state, creating a contention surface for unrelated users.

Source: `programs/[project]/src/instructions/public/position/distribute_fees.rs:21-40`

```rust
#[derive(Accounts)]
pub struct DistributeFees<'info> {
    /// #1
    /// Anyone can call this instruction
    #[account(mut)]
    pub caller: Signer<'info>,

    /// #2
    /// CHECK: Empty PDA, authority for token accounts
    #[account(
        seeds = [b"transfer_authority"],
        bump = cortex.load()?.transfer_authority_bump
    )]
    pub transfer_authority: AccountInfo<'info>,

    /// #3
    #[account(
        seeds = [b"cortex"],
        bump = cortex.load()?.bump,
        constraint = cortex.load()?.is_initialized() @[project]Error::InvalidCortexState,
```

#### Step 2 — Reach the instruction as a third party

The historical PoC explicitly uses a third-party signer and expects success. Sustained denial requires competing transactions and transaction-ordering conditions; successful invocation alone does not prove a guaranteed whole-block lock.

Source: `programs/[project]/tests/[auditor]_tests/l4_distribute_fees_permissionless.rs:191-210`

```rust
    // The call must succeed for the PoC to hold.
    let result = test_instructions::distribute_fees(
        &test_setup.program_test_ctx,
        alice, // ← third-party caller: neither admin nor whitelisted keeper
        &test_setup.payer_keypair,
        &test_setup.pool_pda,
        &DistributeFeesParams {
            oracle_prices: Some(
                utils::get_oracle_prices_as_chaos_labs_bundle(&test_setup.program_test_ctx).await,
            ),
            multi_oracle_prices: None,
        },
    )
    .await;

    assert!(
        result.is_ok(),
        "L-5 PoC: alice's distribute_fees call MUST succeed (permissionless). Error: {:?}",
        result.err()
    );
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/l4_distribute_fees_permissionless.rs`; function `l5_poc_any_caller_can_invoke_distribute_fees`.

The final report labels this Resolved through a zero-debt early return but also retains permissionless fee-present contention and older no-op locking prose. An early return does not change Solana message account writability. The available test does not settle the scheduler-level question. The integration tests model price movement and delays; indirect LP gain is conditional.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/l4_distribute_fees_permissionless.rs:181-210`

```rust
        utils::get_zero_copy_account::<Pool>(&test_setup.program_test_ctx, test_setup.pool_pda)
            .await;

    assert_eq!(pool_before.fees_debt_usd,       0, "L-5 Setup: legacy fees_debt_usd must be 0");
    assert_eq!(pool_before.lm_fee_debt_usd,      0, "L-5 Setup: lm_fee_debt_usd must be 0");
    assert_eq!(pool_before.protocol_fee_debt_usd, 0, "L-5 Setup: protocol_fee_debt_usd must be 0");
    assert_eq!(pool_before.referrers_fee_debt_usd, 0, "L-5 Setup: referrers_fee_debt_usd must be 0");
    assert_eq!(pool_before.manager_fee_debt_usd,  0, "L-5 Setup: manager_fee_debt_usd must be 0");

    // alice is an ordinary user — NOT the pool admin or a designated keeper.
    // The call must succeed for the PoC to hold.
    let result = test_instructions::distribute_fees(
        &test_setup.program_test_ctx,
        alice, // ← third-party caller: neither admin nor whitelisted keeper
        &test_setup.payer_keypair,
        &test_setup.pool_pda,
        &DistributeFeesParams {
            oracle_prices: Some(
                utils::get_oracle_prices_as_chaos_labs_bundle(&test_setup.program_test_ctx).await,
            ),
            multi_oracle_prices: None,
        },
    )
    .await;

    assert!(
        result.is_ok(),
        "L-5 PoC: alice's distribute_fees call MUST succeed (permissionless). Error: {:?}",
        result.err()
    );
```
