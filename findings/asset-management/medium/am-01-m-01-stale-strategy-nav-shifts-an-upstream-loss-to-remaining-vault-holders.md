# Based on deployed contracts

Project: Anonymous asset-management vault protocol (AM-01)  
Category: Asset management, vaults and integrations  
Severity: Medium  
Engagement: Independent hunting + responsible disclosure  
Chain: Stellar / Soroban; Web2 findings concern the associated console and API  
Tech stack: Rust 2021, Soroban SDK 22.0.3, WASM, token/strategy vaults, Blend integration; JavaScript/React, Axios, bearer-token API and XDR signing workflows  
Evidence: Historical local or fork test excerpts; scope and fixture assumptions are described below  
Remediation: not demonstrated  
Contribution: Research and disclosure case study; exclusive discovery credit is not asserted  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Stale strategy NAV shifts an upstream loss to remaining vault holders

## Description

Holder → upstream loss not yet written into strategy NAV → withdraw dfTokens → valuation omits pending loss → holder receives stale claim and remaining shares absorb later haircut.

Historical [project] withdrawal 36.9483620 USDC versus reconstructed 31.7877400; approximately 5.1606220 USDC shifted to remaining holders. No link to upstream attacker; repeated profitable attack not demonstrated.

### Mechanism step by step

#### Step 1 — A strategy continues reporting its pre-loss value

The vault asks the strategy for balance and subtracts locked fees. This does not independently establish the economic recoverability of the reported underlying.

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

#### Step 2 — The holder receives a pro-rata quote using reported funds

The quote multiplies reported total_amount by shares and divides by supply. A hidden upstream impairment therefore survives into the redemption quote.

Source: `source/[project]-1.0.0/apps/contracts/vault/src/lib.rs:331-348`.

```rust
        
        let mut withdrawn_amounts: Vec<i128> = Vec::new(&e);
        
        // Loop through each asset to handle the withdrawal
        for (i, asset) in total_managed_funds.iter().enumerate() {
            // Use assets instead of asset_withdrawal_amounts
            let asset_address = &asset.asset;

            // Calculate the requested withdrawal amount for this asset
            let requested_withdrawal_amount = asset
                .total_amount
                .checked_mul(withdraw_shares)
                .ok_or(ContractError::ArithmeticError)?
                .checked_div(total_shares_supply)
                .ok_or(ContractError::ArithmeticError)?;

            if requested_withdrawal_amount < min_amounts_out.get(i as u32).unwrap() {
                panic_with_error!(&e, ContractError::InsufficientOutputAmount);
```

#### Step 3 — The vault spends liquid assets before unwinding the strategy

In the constructed test, fresh idle pays the inflated exit. In the historical incident the vault had zero idle, and the exit consumed still-realizable dependency pool liquidity. Both shift the unrecognized loss to holders who remain.

Source: `source/[project]-1.0.0/apps/contracts/vault/src/lib.rs:349-368`.

```rust
            }
            if requested_withdrawal_amount > 0 {
                // Process the withdrawal if the requested amount is greater than zero
                let idle_funds = asset.idle_amount;
                if idle_funds >= requested_withdrawal_amount {
                    TokenClient::new(&e, asset_address).transfer(
                        &e.current_contract_address(),
                        &from,
                        &requested_withdrawal_amount,
                    );
                    withdrawn_amounts.push_back(requested_withdrawal_amount);
                } else {
                    if idle_funds != 0 {
                        TokenClient::new(&e, asset_address).transfer(
                            &e.current_contract_address(),
                            &from,
                            &idle_funds,
                        );
                    }
                    let mut cumulative_amount_for_asset = idle_funds;
```

## Proof of concept

The Rust harness loads the deployed vault WASM but constructs a programmable strategy and synthetic balances. It reduces actual strategy backing by 50% while keeping the old reported balance, then supplies fresh idle through a second depositor. Assertions show the early holder receives 99,999,000 raw, the remaining holder 50,000,499 raw, and the early excess over a fair claim is 24,999,750 raw. Separate historical reconstruction identifies a 36.9483620 USDC withdrawal versus 31.7877400 post-loss value, shifting 5.1606220 USDC to remaining holders. The withdrawer is not linked to the upstream attacker.

**Proof limits:** Synthetic impairment model proves accounting composition; current ability to induce an upstream loss is not proved.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `outputs/mgr13-composition-pass-2026-10-01/harnesses/vault-composition/src/lib.rs:517-568`.

```rust
fn stale_reported_strategy_value_lets_early_holder_capture_fresh_idle() {
    let setup = setup(0);
    seed(&setup, 100_000_000);

    // The external position has already lost 50%, but the strategy still reports
    // the old mark. This models Blend before its supplier b_rate write-down.
    setup
        .token_admin
        .burn(&setup.strategy.address, &50_000_000);
    assert_eq!(setup.strategy.balance(&setup.vault.address), 100_000_000);
    assert_eq!(setup.token.balance(&setup.strategy.address), 50_000_000);

    // A later user supplies fresh idle at the stale price per share.
    let victim = Address::generate(&setup.env);
    let victim_shares = deposit(&setup, &victim, 100_000_000, false);
    assert_eq!(victim_shares, 100_000_000);
    assert_eq!(setup.token.balance(&setup.vault.address), 100_000_000);

    // The pre-loss holder owns half the reported NAV. Its entire inflated quote
    // is paid from the victim's good idle, so no strategy withdrawal is attempted.
    let attacker_shares = setup.vault.balance(&setup.initial);
    let attacker_before = setup.token.balance(&setup.initial);
    setup.vault.withdraw(
        &attacker_shares,
        &vec![&setup.env, 0i128],
        &setup.initial,
    );
    let attacker_received = setup.token.balance(&setup.initial) - attacker_before;
    // The first depositor permanently locks 1,000 minimum-liquidity shares.
    assert_eq!(attacker_received, 99_999_000);
    assert_eq!(setup.token.balance(&setup.vault.address), 1_000);

    // Once the external loss is recognized, the remaining holder receives only
    // the realizable 50m. At impairment time the true pro-rata claims were 75m
    // each, so the early holder captured 25m of the victim's value.
    set_strategy_value(&setup, 50_000_000);
    let victim_before = setup.token.balance(&victim);
    setup.vault.withdraw(
        &victim_shares,
        &vec![&setup.env, 0i128],
        &victim,
    );
    let victim_received = setup.token.balance(&victim) - victim_before;
    assert_eq!(victim_received, 50_000_499);
    let attacker_true_claim_at_impairment = 74_999_250;
    assert_eq!(attacker_received - attacker_true_claim_at_impairment, 24_999_750);
    assert_eq!(75_000_000 - victim_received, 24_999_501);
    println!(
        "stale NAV capture: early={} remaining={} shifted={}",
        attacker_received,
        victim_received,
        attacker_received - attacker_true_claim_at_impairment,
```
