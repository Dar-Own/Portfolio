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

# Public invest-on-deposit deletes negative strategy loss carry

## Description

Depositor → negative strategy carry exists → deposit(..., invest=true) → investment assigns new external-flow-adjusted delta instead of accumulating old carry → negative carry erased, later recovery charged as profit.

Exactly 0.5607991 USDC historical improper fees. Exact-WASM control: -200 carry erased and 40 fee on 200 recovery versus zero control fee; latest reachable value dust-sized.

### Mechanism step by step

#### Step 1 — Public investment follows the depositor-selected flag

Only the depositor authenticates the ordinary deposit. The invest flag routes fresh funds into the strategy accounting path.

Source: `source/[project]-1.0.0/apps/contracts/vault/src/lib.rs:231-275`.

```rust
    fn deposit(
        e: Env,
        amounts_desired: Vec<i128>,
        amounts_min: Vec<i128>,
        from: Address,
        invest: bool,
    ) -> Result<(Vec<i128>, i128, Option<Vec<Option<AssetInvestmentAllocation>>>), ContractError> {
        extend_instance_ttl(&e);
        from.require_auth();

        // Fetches the total managed funds for all assets, including idle and invested funds (net of locked fees).
        // Setting the flag to `true` ensures that strategy reports are updated and new fees are locked during the process.
        let total_managed_funds = fetch_total_managed_funds(&e, true)?;

        let (amounts, shares_to_mint, total_shares_supply) = process_deposit(
            &e,
            &total_managed_funds,
            &amounts_desired,
            &amounts_min,
            &from,
        )?;
        events::emit_deposit_event(&e, from, amounts.clone(), shares_to_mint.clone(), total_shares_supply, total_managed_funds.clone());

        let asset_investments = if invest {
            let allocations = generate_investment_allocations(&e, &total_managed_funds, &amounts)?;
            //iterate between allocations to execute investments
            for allocation in allocations.iter() {
                //Validate if alloctation is not empty
                if let Some(allocation) = allocation.clone() {
                    let strategies = allocation.strategy_allocations;
                    for strategy in strategies {
                        if let Some(strategy) = strategy {
                            let strategy_address = strategy.strategy_address;
                            let amount = strategy.amount;
                            invest_in_strategy(&e, &allocation.asset, &strategy_address, &amount)?;
                        }

                    }
                }
            }
            Some(allocations)
        } else {
            None
        };
        Ok((amounts, shares_to_mint, asset_investments))
```

#### Step 2 — Ordinary reports accumulate negative carry

The normal clock adds current minus previous balance to the accumulated gains_or_losses.

Source: `source/[project]-1.0.0/apps/contracts/vault/src/report.rs:115-136`.

```rust
    pub fn report(&mut self, current_balance: i128) -> Result<(), ContractError> {
        // Use current balance as previous balance if this is the first report (prev_balance is 0)
        let prev_balance = if self.prev_balance == 0 {
            current_balance
        } else {
            self.prev_balance
        };

        // Calculate gains or losses and handle potential underflow
        let current_gains_or_losses = current_balance
            .checked_sub(prev_balance)
            .ok_or(ContractError::Underflow)?;

        // Update the accumulated gains or losses, handle overflow
        self.gains_or_losses = self.gains_or_losses
            .checked_add(current_gains_or_losses)
            .ok_or(ContractError::Overflow)?;

        // Update the previous balance for the next report
        self.prev_balance = current_balance;

        Ok(())
```

#### Step 3 — Investment replaces that carry with the external-flow-adjusted delta

The investment clock assigns its computed delta rather than adding it. An honest external deposit can replace an existing loss carry with approximately zero.

Source: `source/[project]-1.0.0/apps/contracts/vault/src/strategies.rs:169-180`.

```rust

    // Reports
    // Store Strategy invested funds for reports
    report.gains_or_losses = strategy_funds
        .checked_sub(report.prev_balance)
        .unwrap_or_else(|| panic_with_error!(e, ContractError::ArithmeticError))
        .checked_sub(*amount)
        .unwrap_or_else(|| panic_with_error!(e, ContractError::ArithmeticError));
    report.prev_balance = strategy_funds;
    set_report(e, strategy_address, &report);

    Ok(report)
```

#### Step 4 — Recovery is then feeable before the old loss is recovered

The next report sees positive gains. Fee locking consumes those gains even when economically they merely restore the previously erased loss.

Source: `source/[project]-1.0.0/apps/contracts/vault/src/report.rs:44-62`.

```rust
    /// * `Result<(), ContractError>` - Returns `Ok(())` on success, or a `ContractError::Overflow`
    ///   if an arithmetic overflow occurs during fee calculation or locked fee update.
    /// 
    /// # Examples
    /// If `gains_or_losses = 50`, `fee_rate = 1000` (10%), and `SCALAR_BPS = 10,000`, 
    /// then `total_fee = (50 * 1000) / 10,000 = 5`. This fee is added to `locked_fee`, 
    /// and `gains_or_losses` is reset to 0.
    pub fn lock_fee(&mut self, fee_rate: u32) -> Result<(), ContractError> {
        if self.gains_or_losses <= 0 {
            return Ok(());
        }
        let gains_or_losses = self.gains_or_losses;
        let numerator = gains_or_losses.checked_mul(fee_rate as i128).unwrap();
        let total_fee = numerator.checked_div(SCALAR_BPS).unwrap();

        self.locked_fee = self.locked_fee.checked_add(total_fee).ok_or(ContractError::Overflow)?;
        self.gains_or_losses = 0;
        Ok(())
    }
```

## Proof of concept

One local test reduces the synthetic strategy value from 100,000,000 to 90,000,000 raw and checkpoints a −10,000,000 loss. A one-raw-unit deposit with invest=true resets carry to zero; recovery then locks 1,000,000 raw at its 10% fee. A separate exact-WASM candidate/control uses 1,000 initial tokens, 200 loss, 100 deposit and 20% fees: the candidate charges 40 while the invest=false control charges zero. These are different fixtures. Historical event reconstruction attributes 0.5607991 USDC improper fees; the later exposed amount was dust-sized.

**Proof limits:** Different fixture fee rates are kept separate; current materiality does not support a deployed High claim.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `outputs/mgr13-composition-pass-2026-10-01/harnesses/vault-composition/src/lib.rs:411-427`.

```rust
fn invest_deposit_erases_negative_loss_carry_then_recovery_is_charged_again() {
    let setup = setup(1_000);
    seed(&setup, 100_000_000);

    set_strategy_value(&setup, 90_000_000);
    setup.vault.get_asset_amounts_per_shares(&0);
    assert_eq!(report(&setup).gains_or_losses, -10_000_000);

    let attacker = Address::generate(&setup.env);
    // One raw unit is sufficient in this exact-vault composition.
    deposit(&setup, &attacker, 1, true);
    assert_eq!(report(&setup).gains_or_losses, 0);

    add_strategy_value(&setup, 10_000_000);
    setup.vault.get_asset_amounts_per_shares(&0);
    assert_eq!(report(&setup).locked_fee, 1_000_000);
}
```
