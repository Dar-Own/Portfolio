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

# Account revival: closed position PDAs can be zombified

## Description

Keeper closes a triggered SL/TP position and bundles a lamport transfer back to the PDA in the same transaction; manual close zeroes fields but leaves Anchor discriminator, preventing garbage collection and reinitialization.

Permanent victim position-slot lockout in tested version for rent-minimum cost; bump mismatch blocks profit extraction from zombie itself.

The local integration test constructs accounts, opens a position and bundles closing plus rent restoration. The asserted zombie has a valid discriminator and zero bump/owner/size. No direct extraction from the zombie is established. The final report records discriminator clearing plus updated Anchor initialization handling as the remediation.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Drain rent and reset only the position fields

A keeper can close an eligible triggered position. Manual lamport removal and Position::default reset the struct but leave the Anchor discriminator outside that struct intact.

Source: `programs/[project]/src/instructions/public/position/close_position_long.rs:615-625`

```rust
    } else {
        // Delete the position
        Cortex::transfer_sol_from_owned(
            ctx.accounts.position.to_account_info(),
            ctx.accounts.owner.to_account_info(),
            ctx.accounts.position.get_lamports(),
        )?;

        // Deallocate the position account
        *position = Position::default();
    }
```

#### Step 2 — Revive the account before transaction completion

The attack bundles a lamport transfer back into the closing transaction. The surviving discriminator permits deserialization, while zeroed owner/bump fields block normal operations and historical init refuses the existing account.

Source: `programs/[project]/tests/[auditor]_tests/m4_account_revival_position_dos.rs:698-720`

```rust
    // Step 5: Verify the zombie state
    {
        let mut ctx = ts.program_test_ctx.write().await;
        let raw = ctx.banks_client.get_account(position_pda).await.unwrap().unwrap();

        // Discriminator is STILL valid
        assert_eq!(
            &raw.data[..8],
            &<Position as anchor_lang::Discriminator>::DISCRIMINATOR,
            "Zombie must retain valid Position discriminator"
        );

        // But bump is 0
        // Note: we need to drop ctx first since try_get_zero_copy_account takes the lock
        drop(ctx);

        let zombie_pos = utils::try_get_zero_copy_account::<Position>(&ts.program_test_ctx, position_pda).await;
        assert!(zombie_pos.is_some(), "Zombie must be deserializable");
        let zombie = zombie_pos.unwrap();
        assert_eq!(zombie.bump, 0, "Zombie bump must be 0");
        assert_eq!(zombie.owner, Pubkey::default(), "Zombie owner must be default");
        assert_eq!(zombie.size_usd, 0, "Zombie size must be 0");
        assert_eq!(zombie.collateral_usd, 0, "Zombie collateral must be 0");
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/m4_account_revival_position_dos.rs`; function `integ_competitive_griefing_blocks_position_reopening`.

The local integration test constructs accounts, opens a position and bundles closing plus rent restoration. The asserted zombie has a valid discriminator and zero bump/owner/size. No direct extraction from the zombie is established. The final report records discriminator clearing plus updated Anchor initialization handling as the remediation.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/m4_account_revival_position_dos.rs:698-720`

```rust
    // Step 5: Verify the zombie state
    {
        let mut ctx = ts.program_test_ctx.write().await;
        let raw = ctx.banks_client.get_account(position_pda).await.unwrap().unwrap();

        // Discriminator is STILL valid
        assert_eq!(
            &raw.data[..8],
            &<Position as anchor_lang::Discriminator>::DISCRIMINATOR,
            "Zombie must retain valid Position discriminator"
        );

        // But bump is 0
        // Note: we need to drop ctx first since try_get_zero_copy_account takes the lock
        drop(ctx);

        let zombie_pos = utils::try_get_zero_copy_account::<Position>(&ts.program_test_ctx, position_pda).await;
        assert!(zombie_pos.is_some(), "Zombie must be deserializable");
        let zombie = zombie_pos.unwrap();
        assert_eq!(zombie.bump, 0, "Zombie bump must be 0");
        assert_eq!(zombie.owner, Pubkey::default(), "Zombie owner must be default");
        assert_eq!(zombie.size_usd, 0, "Zombie size must be 0");
        assert_eq!(zombie.collateral_usd, 0, "Zombie collateral must be 0");
```
