# Based on deployed contracts

Project: Anonymous asset-management vault protocol (AM-01)  
Category: Asset management, vaults and integrations  
Severity: Medium  
Engagement: Independent hunting + responsible disclosure  
Chain: Stellar / Soroban; Web2 findings concern the associated console and API  
Tech stack: Rust 2021, Soroban SDK 22.0.3, WASM, token/strategy vaults, Blend integration; JavaScript/React, Axios, bearer-token API and XDR signing workflows  
Evidence: Historical observation or unsigned simulation, with supporting source/test excerpts  
Remediation: not demonstrated  
Contribution: Research and disclosure case study; exclusive discovery credit is not asserted  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Incompatible fee receiver blocks management and incident response

## Description

Unprivileged caller without shares or a deposit → strategy has positive uncheckpointed gains and the configured fee receiver lacks the asset’s required trustline → calls get_asset_amounts_per_shares → quotation persists positive locked fees → legitimate operator later invokes rescue, rebalance or fee distribution → synchronous USDC fee transfer fails at the receiver and atomically reverts the safety operation.

Fee-paying management unavailable; ordinary withdrawal remained available in tested configuration. Theft and general withdrawal freeze not demonstrated.

### Mechanism step by step

#### Step 1 — An outsider triggers accounting through a quote

The quote has no caller-authentication argument and passes true to the fee-checkpoint path.

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

#### Step 2 — The checkpoint persists a positive fee

The report update and lock_fee are written to storage. The attacker needs positive accrued yield, not a management role.

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

#### Step 3 — An authorized rescue synchronously distributes that fee

The legitimate operator passes its role check but fee distribution precedes the emergency unwind.

Source: `source/[project]-1.0.0/apps/contracts/vault/src/lib.rs:431-456`.

```rust
    fn rescue(
        e: Env,
        strategy_address: Address,
        caller: Address,
    ) -> Result<(), ContractError> {
        extend_instance_ttl(&e);

        // Ensure the caller is the Manager or Emergency Manager
        let access_control = AccessControl::new(&e);
        access_control.require_any_role(
            &[RolesDataKey::EmergencyManager, RolesDataKey::Manager],
            &caller,
        );

        // Find the strategy and its associated asset
        let asset = get_strategy_asset(&e, &strategy_address)?;
        // This ensures that the vault has this strategy in its list of assets
        let strategy = get_strategy_struct(&strategy_address, &asset)?;
        let strategy_invested_funds = fetch_strategy_invested_funds(&e, &strategy_address, true)?;
        let distribution_result = report::distribute_strategy_fees(&e, &strategy.address, &access_control, &asset.address)?;
        if distribution_result > 0 {
            let mut distributed_fees: Vec<(Address, i128)> = Vec::new(&e);
            distributed_fees.push_back((asset.address.clone(), distribution_result));
            events::emit_fees_distributed_event(&e, distributed_fees.clone());
        }

```

#### Step 4 — Receiver rejection rolls back the entire operation

The transfer to an incompatible fee receiver fails. Soroban transaction atomicity rolls back the management operation and leaves the preexisting locked fee in place. The simulation identifies a missing asset trustline; ordinary user withdrawals do not use this payout path.

Source: `source/[project]-1.0.0/apps/contracts/vault/src/report.rs:189-219`.

```rust
pub fn distribute_strategy_fees(e: &Env, strategy_address: &Address, access_control: &AccessControl, asset: &Address) -> Result<i128, ContractError> {
    let mut report = get_report(e, strategy_address);
    
    let [project]_fee = get_[project]_protocol_fee_rate(&e);
    let [project]_protocol_receiver = get_[project]_protocol_fee_receiver(&e)?;
    let vault_fee_receiver = access_control.get_fee_receiver()?;

    let fees_to_distribute = report.locked_fee;

    if fees_to_distribute > 0 {
        // Calculate shares for each receiver based on their fee proportion
        let numerator = fees_to_distribute.checked_mul([project]_fee as i128).unwrap();
        let [project]_fee_amount = numerator.checked_div(SCALAR_BPS).unwrap();

        let vault_fee_amount = fees_to_distribute.checked_sub([project]_fee_amount).ok_or(ContractError::Underflow)?;

        let remaining_balance = unwind_from_strategy(
            &e,
            &strategy_address,
            &fees_to_distribute,
            &e.current_contract_address(),
        )?;

        // Transfer fees to the respective receivers
        let asset_client = TokenClient::new(&e, &asset);
        asset_client.transfer( &e.current_contract_address(), &vault_fee_receiver, &vault_fee_amount);
        asset_client.transfer( &e.current_contract_address(), &[project]_protocol_receiver, &[project]_fee_amount);

        report.prev_balance = remaining_balance;
        report.locked_fee = 0;
        set_report(&e, &strategy_address, &report);
```

## Proof of concept

Unsigned, non-submitted production simulations of distribute_fees and rescue_vault reached a missing-trustline failure. They attempted a 272636758764-raw fee unwind and 218109407012-raw receiver transfer. Because the transaction failed, diagnostic events do not represent committed transfers. The test excerpt below is the compatible-receiver control: it shows successful rescue and payout when the receiver can accept the asset. It is not an executable reproduction of the incompatible receiver itself.

**Proof limits:** Primary adverse result is an unsigned simulation; local test is a successful control, not the failing deployment reproduction.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `outputs/mgr13-composition-pass-2026-10-01/harnesses/vault-composition/src/lib.rs:224-245`.

```rust
fn rescue_succeeds_when_fee_receiver_can_receive_asset() {
    let setup = setup(1_000);
    seed(&setup, 100_000_000);
    add_strategy_value(&setup, 10_000_000);
    setup.vault.get_asset_amounts_per_shares(&0);
    assert_eq!(report(&setup).locked_fee, 1_000_000);

    setup.vault.rescue(&setup.strategy.address, &setup.manager);

    assert_eq!(setup.token.balance(&setup.fee_receiver), 1_000_000);
    assert_eq!(setup.token.balance(&setup.vault.address), 109_000_000);
    assert_eq!(setup.strategy.balance(&setup.vault.address), 0);
    assert_eq!(
        report(&setup),
        vault::Report {
            gains_or_losses: 0,
            locked_fee: 0,
            prev_balance: 0,
        }
    );
    assert!(
        setup
```
