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

# Griefing: Deprecated Permissionless IDL

## Description

Funded caller invokes deprecated genesis_stake_patch exposed in IDL; no-op returns Ok while mutable staking/cortex/vault accounts still contend for writes.

Griefing and misleading successful integration response; no profit path established.

The test targets another user with an unrelated signer and expects success. It does not establish a guaranteed indefinite lock under real transaction scheduling or profit. The final report says the handler was changed to InvalidInstructionData and records Resolved.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Supply an unrelated caller and victim owner

The account schema permits any caller and explicitly accepts any owner wallet; mutable staking-related accounts can therefore be targeted by an unrelated funded sender.

Source: `programs/[project]/src/instructions/public/staking_p/genesis_stake_patch.rs:14-26`

```rust
#[derive(Accounts)]
pub struct GenesisStakePatch<'info> {
    /// #1
    pub caller: Signer<'info>,

    /// #2
    // Pay for realloc
    #[account(mut)]
    pub payer: Signer<'info>,

    /// #3
    /// CHECK: Can be any wallet
    pub owner: AccountInfo<'info>,
```

#### Step 2 — Return success from a deprecated no-op

The historic handler body is commented out and returns Ok. The ABI remains callable despite no meaningful work, creating misleading integration success and a contention surface.

Source: `programs/[project]/src/instructions/public/staking_p/genesis_stake_patch.rs:340-345`

```rust
    //         staking.resolved_lm_reward_token_amount -= lm_rewards_token_amount;
    //     }
    // }

    Ok(())
}
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/l10_genesis_stake_patch_permissionless.rs`; function `l6_poc_any_caller_can_invoke_genesis_stake_patch`.

The test targets another user with an unrelated signer and expects success. It does not establish a guaranteed indefinite lock under real transaction scheduling or profit. The final report says the handler was changed to InvalidInstructionData and records Resolved.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/l10_genesis_stake_patch_permissionless.rs:208-227`

```rust
    // bob calls genesis_stake_patch targeting alice's user_staking.
    // bob is NOT alice, NOT the admin, and has no relationship to alice's account.
    // The accounts struct accepts this because caller has no constraint and
    // owner is CHECK-annotated as "Can be any wallet".
    let result = test_instructions::genesis_stake_patch(
        &test_setup.program_test_ctx,
        bob,   // ← caller: unrelated third party
        alice, // ← owner: the victim whose user_staking is targeted
        &test_setup.payer_keypair,
        &test_setup.pool_pda,
        &lp_token_mint_pda,
    )
    .await;

    assert!(
        result.is_ok(),
        "L-6 PoC: bob's genesis_stake_patch call targeting alice MUST succeed \
         (permissionless). Error: {:?}",
        result.err()
    );
```
