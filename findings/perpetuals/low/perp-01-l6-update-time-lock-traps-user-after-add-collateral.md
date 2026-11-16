# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Low — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Original local test source and historical report; mechanism assertions shown below
Remediation: Acknowledged — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Update-Time Lock Traps User After add_collateral

## Description

Position owner adds collateral; blanket mutation resets update_time though size/PnL is unchanged, so close_position fails min_position_update_time_before_close_seconds (default 120s).

Owner temporarily cannot close while market moves against the position; project accepts lock as stale-price protection.

The integration test first warps past the separate open-time lock, adds collateral, advances one second and expects PositionTooYoung. Companion controls close successfully after the update window and show that a second top-up resets it. The final response accepts the delay as stale-price protection.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Reset the exit clock when the owner adds collateral

A protective collateral top-up writes update_time even though it does not increase position size or directly change PnL.

Source: `programs/[project]/src/instructions/public/position/add_collateral_long.rs:210-220`

```rust
        }
    }

    // Update virtual funding rate (updates stored cumulative values)
    custody.update_virtual_funding_rate(current_time)?;

    // Update existing position
    {
        position.update_time = current_time;
        position.collateral_usd += collateral_usd;
        position.collateral_amount += params.collateral;
```

#### Step 2 — Reject a close inside the new update-time window

The close path checks elapsed time since update_time. The owner cannot immediately reverse the exposure after topping up; the reported default is 120 seconds.

Source: `programs/[project]/src/instructions/public/position/close_position_long.rs:216-229`

```rust
        require!(
            current_time
                >= position.open_time + pool_exit_fee_config.min_position_open_time_seconds as i64,
            [project]Error::PositionTooYoung
        );
        // Check that the position wasn't updated in the last configured minimum time
        require!(
            current_time
                >= position.update_time
                    + pool_exit_fee_config.min_position_update_time_before_close_seconds as i64,
            [project]Error::PositionTooYoung
        );

        // Price cannot be 0, as 0 is used to indicate no price limit internally
```

## Proof of concept

Test: `programs/[project]/tests/[auditor]_tests/l6_integ_time_lockout_enforcement.rs`; function `close_rejected_immediately_after_add_collateral`.

The integration test first warps past the separate open-time lock, adds collateral, advances one second and expects PositionTooYoung. Companion controls close successfully after the update window and show that a second top-up resets it. The final response accepts the delay as stale-price protection.

The excerpts show the original test assertions. Local funding, position construction, oracle inputs and any privileged setup are fixtures; they do not establish those conditions on a live deployment. The tests require their matching historical source revision, and repaired code may correctly reject the exploit expectation.

## Relevant test excerpts

Source: `programs/[project]/tests/[auditor]_tests/l6_integ_time_lockout_enforcement.rs:243-263`

```rust
async fn close_rejected_immediately_after_add_collateral() {
    let ts = setup().await;
    let position_pda = open_eth_long(&ts).await;

    // Warp past the open_time lock so only the update lock matters.
    utils::warp_forward(&ts.program_test_ctx, OPEN_TIME_LOCK as i64 + 1).await;

    // add_collateral sets update_time = now.
    add_collateral(&ts, &position_pda).await;

    // Immediately try to close — should fail because we're within the
    // update-time lockout window.
    utils::warp_forward(&ts.program_test_ctx, 1).await;

    let result = try_close(&ts, &position_pda).await;
    assert!(
        result.is_err(),
        "Close should fail: only 1s elapsed since add_collateral, lockout is {UPDATE_TIME_LOCK}s"
    );
    assert_[project]_error(result.unwrap_err(), [project]Error::PositionTooYoung);
}
```
