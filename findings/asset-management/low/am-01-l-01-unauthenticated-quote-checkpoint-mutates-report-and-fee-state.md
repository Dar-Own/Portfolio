# Based on deployed contracts

Project: Anonymous asset-management vault protocol (AM-01)  
Category: Asset management, vaults and integrations  
Severity: Low  
Engagement: Independent hunting + responsible disclosure  
Chain: Stellar / Soroban; Web2 findings concern the associated console and API  
Tech stack: Rust 2021, Soroban SDK 22.0.3, WASM, token/strategy vaults, Blend integration; JavaScript/React, Axios, bearer-token API and XDR signing workflows  
Evidence: Historical local or fork test excerpts; scope and fixture assumptions are described below  
Remediation: not demonstrated  
Contribution: Research and disclosure case study; exclusive discovery credit is not asserted  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Unauthenticated quote/checkpoint mutates report and fee state

## Description

Unauthenticated caller → positive strategy gains or a pending fee-rate transition → invokes get_asset_amounts_per_shares, including a zero-share quote → its update flag checkpoints strategy reports and locks fees despite the quotation interface → an outsider chooses the fee-accounting time.

Local holder outcome shifted about one token on ten-token gain; live simulated target only about $0.00218 and fee-lock headroom 10664 raw USDC.

### Mechanism step by step

#### Step 1 — A quote-like call enters the mutable accounting path

A zero-share quote can return zero while still checkpointing every strategy because the true flag is independent of the requested share count.

Source: `source/[project]-1.0.0/apps/contracts/vault/src/lib.rs:586-600`.

```rust
    fn get_asset_amounts_per_shares(
        e: Env,
        vault_shares: i128,
    ) -> Result<Vec<i128>, ContractError> {
        extend_instance_ttl(&e);

        // Fetches the total managed funds for all assets, including idle and invested funds (net of locked fees).
        // Setting the flag to `true` ensures that strategy reports are updated and new fees are locked during the process.
        let total_managed_funds = fetch_total_managed_funds(&e, true)?;
        Ok(calculate_asset_amounts_per_vault_shares(
            &e,
            vault_shares,
            &total_managed_funds,
        )?)
    }
```

#### Step 2 — The checkpoint locks fees at the current rate

The caller selects timing relative to a later manager fee change. This changes which gains are charged at each rate; it does not give the outsider permission to set the fee.

Source: `source/[project]-1.0.0/apps/contracts/vault/src/report.rs:175-186`.

```rust
pub fn update_report_and_lock_fees(
    e: &Env, 
    strategy_address: &Address,
    strategy_invested_funds: i128
) -> Result<Report, ContractError> {

    let mut report = get_report(&e, &strategy_address);
    report.report(strategy_invested_funds)?;
    report.lock_fee(get_vault_fee(&e))?;
    set_report(&e, &strategy_address, &report);

    Ok(report)
```

#### Step 3 — Locked fees reduce investable value and can approach a freeze boundary

The current locked fee is compared to strategy backing. The observed dust-sized exposure and need for a later loss constrain the liveness composition.

Source: `source/[project]-1.0.0/apps/contracts/vault/src/funds.rs:45-62`.

```rust
pub fn fetch_strategy_invested_funds(e: &Env, strategy_address: &Address, report_and_lock_fees: bool) -> Result<i128, ContractError> {
    let strategy_client = get_strategy_client(e, strategy_address.clone());
    let strategy_invested_funds = strategy_client.balance(&e.current_contract_address());
    
    if !report_and_lock_fees {
        let locked_fee = get_report(e, strategy_address).locked_fee;
        if locked_fee > strategy_invested_funds {
            return Err(ContractError::WrongLockedFees);
        }
        return Ok(strategy_invested_funds
            .checked_sub(locked_fee).unwrap_or_else(|| panic_with_error!(&e, ContractError::ArithmeticError)))
    } else {
        let locked_fee = report::update_report_and_lock_fees(e, strategy_address, strategy_invested_funds)?.locked_fee;
        if locked_fee > strategy_invested_funds {
            return Err(ContractError::WrongLockedFees);
        }
        Ok(strategy_invested_funds.checked_sub(locked_fee).unwrap_or_else(|| panic_with_error!(&e, ContractError::ArithmeticError)))
    }
```

## Proof of concept

The exact-vault local harness uses a synthetic strategy and test-generated actors. The zero-share quote test asserts that a zero result still persists a positive fee. Candidate/control fee-ordering tests compare manager-first and outsider-first updates and assert a 999,990-raw holder-exit difference. A deployment simulation separately records locked_fee 896162 and 10664 raw backing headroom; a subsequent 10665-raw loss would cross the freeze threshold. That later loss is a conditional scenario, not an observed event.

**Proof limits:** Fixture raw units and deployment amounts are separate; current exposure was dust-sized.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `outputs/mgr13-composition-pass-2026-10-01/harnesses/vault-composition/src/lib.rs:198-220`.

```rust
fn outsider_quote_zero_commits_fee_checkpoint() {
    let setup = setup(1_000);
    seed(&setup, 100_000_000);
    println!(
        "after seed report={:?} fees={:?}",
        report(&setup),
        setup.vault.get_fees()
    );
    add_strategy_value(&setup, 10_000_000);
    println!(
        "after yield strategy_balance={}",
        setup.strategy.balance(&setup.vault.address)
    );

    assert_eq!(
        setup.vault.get_asset_amounts_per_shares(&0).get(0).unwrap(),
        0
    );
    let after = report(&setup);
    println!("after quote report={after:?}");
    assert_eq!(after.gains_or_losses, 0);
    assert_eq!(after.locked_fee, 1_000_000);
    assert_eq!(after.prev_balance, 110_000_000);
```

Source: `outputs/mgr13-composition-pass-2026-10-01/harnesses/vault-composition/src/lib.rs:388-407`.

```rust
fn outsider_checkpoint_changes_holder_exit_for_fee_increase_and_decrease() {
    let increase_control = fee_ordering_holder_exit(1_000, 2_000, false);
    let increase_outsider = fee_ordering_holder_exit(1_000, 2_000, true);

    let decrease_control = fee_ordering_holder_exit(2_000, 1_000, false);
    let decrease_outsider = fee_ordering_holder_exit(2_000, 1_000, true);
    println!(
        "increase control={increase_control:?} outsider={increase_outsider:?}; decrease control={decrease_control:?} outsider={decrease_outsider:?}",
    );

    assert_eq!(
        (increase_control.0, increase_outsider.0),
        (2_000_000, 1_000_000)
    );
    assert_eq!(increase_outsider.1 - increase_control.1, 999_990);
    assert_eq!(
        (decrease_control.0, decrease_outsider.0),
        (1_000_000, 2_000_000)
    );
    assert_eq!(decrease_control.1 - decrease_outsider.1, 999_990);
```
