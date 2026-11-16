# Source-code-based

Project: Anonymous asset-management vault protocol (AM-01)  
Category: Asset management, vaults and integrations  
Severity: High (future severity)  
Engagement: Independent hunting + responsible disclosure  
Chain: Stellar / Soroban; Web2 findings concern the associated console and API  
Tech stack: Rust 2021, Soroban SDK 22.0.3, WASM, token/strategy vaults, Blend integration; JavaScript/React, Axios, bearer-token API and XDR signing workflows  
Evidence: Partial test or synthetic model; does not establish the complete deployed impact  
Remediation: not demonstrated  
Contribution: Research and disclosure case study; exclusive discovery credit is not asserted  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Spot-reserve valuation permits modeled extraction of idle liquidity

## Description

dfToken holder → functionally funded Xycloans position and idle assets → manipulate pair reserves then redeem and reverse swap → same-ledger caller-controlled spot NAV accepted → inflated claim consumes good idle assets.

Economic model: approximately 57581.5070025 USDC net profit for stated synthetic reserves/capital. No deployed exposure; exact funding fails authorization; a complete exploit against a deployed adapter has not been demonstrated.

### Mechanism step by step

#### Step 1 — The adapter quotes its position against current reserves

A permissionless trade changes the same-ledger reserves used to value the unchanged volatile position.

Source: `source/[project]-1.0.0/apps/contracts/strategies/xycloans/src/lib.rs:121-152`.

```rust
    fn balance(e: Env, from: Address) -> Result<i128, StrategyError> {
        let xycloans_address = get_xycloans_pool_address(&e);
        let xycloans_pool_client = XycloansPoolClient::new(&e, &xycloans_address);

        let shares: i128 = xycloans_pool_client.shares(&from);
        let matured: i128 = xycloans_pool_client.matured(&from);

        let total: i128 = shares.checked_add(matured).unwrap();

        // If total is zero, return it
        if total == 0 {
            return Ok(total);
        }

        // XLM TO USDC QUOTE from SOROSWAP
        let soroswap_factory = get_soroswap_factory_address(&e);
        let pool_token = get_pool_token(&e);
        let token_in = get_token_in(&e);

        // Setting up Soroswap router client
        let (reserve_0, reserve_1) = get_reserves(
            e.clone(),
            soroswap_factory.clone(),
            pool_token.clone(),
            token_in.clone(),
        )
        .map_err(|_| StrategyError::ProtocolAddressNotFound)?;

        let amount_out = get_amount_out(total, reserve_0, reserve_1)
            .map_err(|_| StrategyError::ExternalError)?;

        Ok(amount_out)
```

#### Step 2 — The reserve formula amplifies the manipulated quote

The constant-product quote uses the live reserve ratio with a trading fee, without an independent price or time-weighted bound.

Source: `source/[project]-1.0.0/apps/contracts/strategies/xycloans/src/soroswap_router.rs:131-152`.

```rust
pub fn get_amount_out(
    amount_in: i128,
    reserve_in: i128,
    reserve_out: i128,
) -> Result<i128, SoroswapLibraryError> {
    if amount_in <= 0 {
        return Err(SoroswapLibraryError::InsufficientInputAmount);
    }
    if reserve_in <= 0 || reserve_out <= 0 {
        return Err(SoroswapLibraryError::InsufficientLiquidity);
    }

    let fee = (amount_in.checked_mul(3).unwrap())
        .checked_ceiling_div(1000)
        .unwrap();

    let amount_in_less_fee = amount_in.checked_sub(fee).unwrap();
    let numerator = amount_in_less_fee.checked_mul(reserve_out).unwrap();

    let denominator = reserve_in.checked_add(amount_in_less_fee).unwrap();

    Ok(numerator.checked_div(denominator).unwrap())
```

#### Step 3 — An inflated share redemption fits into idle liquidity

The attacker chooses a share fraction whose inflated quote remains idle-covered, avoiding the broken strategy-withdrawal branch, then reverses the AMM trade. One modeled execution is bounded by available idle.

Source: `source/[project]-1.0.0/apps/contracts/vault/src/lib.rs:339-359`.

```rust
            // Calculate the requested withdrawal amount for this asset
            let requested_withdrawal_amount = asset
                .total_amount
                .checked_mul(withdraw_shares)
                .ok_or(ContractError::ArithmeticError)?
                .checked_div(total_shares_supply)
                .ok_or(ContractError::ArithmeticError)?;

            if requested_withdrawal_amount < min_amounts_out.get(i as u32).unwrap() {
                panic_with_error!(&e, ContractError::InsufficientOutputAmount);
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
```

## Proof of concept

The executable artifact is an integer economic model. It assumes 1m/1m reserves, a 500k volatile-asset position, 100k idle and 1m temporary manipulation capital. The model reports normal valuation 332,665.9993326, manipulated valuation 997,747.1862378, fair share value 39,413.9930173, round-trip cost 3,004.4999799 and modeled net gain 57,581.5070025 USDC. This is not an onchain exploit or a full integration test. The exact adapter’s authorization mismatch blocks funding; atomic production resource fit and affected deployed exposure are not established.

**Proof limits:** Economic model only; hypothetical deployment with repaired funding and sufficient idle required.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `outputs/amm-adapter-composition-2026-10-02/scripts/xycloans_idle_drain_model.mjs:13-24`.

```javascript
function getAmountOut(amountIn, reserveIn, reserveOut) {
  const fee = ceilDiv(amountIn * 3n, 1000n);
  const amountInLessFee = amountIn - fee;
  return (amountInLessFee * reserveOut) / (reserveIn + amountInLessFee);
}

function runCase(vaultFeeBps) {
  const reserveXlm0 = 1_000_000n * SCALE;
  const reserveUsdc0 = 1_000_000n * SCALE;
  const strategyXlm = 500_000n * SCALE;
  const vaultIdleUsdc = 100_000n * SCALE;
  const attackerSwapUsdc = 1_000_000n * SCALE;
```

Source: `outputs/amm-adapter-composition-2026-10-02/scripts/xycloans_idle_drain_model.mjs:45-68`.

```javascript
  // Conservative fee treatment: assume the entire apparent manipulation gain is
  // eligible for the vault performance fee. The actual v1 report state can charge
  // less because the adapter's deposit() returns zero.
  const apparentGain = manipulatedStrategyQuote - fairStrategyQuote;
  const lockedFee = (apparentGain * BigInt(vaultFeeBps)) / BPS;
  const netManipulatedStrategyQuote = manipulatedStrategyQuote - lockedFee;
  const manipulatedNav = vaultIdleUsdc + netManipulatedStrategyQuote;

  // Maximum share fraction whose quoted withdrawal is satisfied entirely from idle.
  // Stored as a 1e12 fixed-point fraction to keep the model deterministic.
  const FRACTION_SCALE = 1_000_000_000_000n;
  const attackerShare = (vaultIdleUsdc * FRACTION_SCALE) / manipulatedNav;
  const quotedPayout = (manipulatedNav * attackerShare) / FRACTION_SCALE;
  const fairNav = vaultIdleUsdc + fairStrategyQuote;
  const fairClaim = (fairNav * attackerShare) / FRACTION_SCALE;

  const recoveredUsdc = getAmountOut(
    attackerXlm,
    reserveXlm1,
    reserveUsdc1,
  );
  const manipulationCost = attackerSwapUsdc - recoveredUsdc;
  const grossVaultOverpayment = quotedPayout - fairClaim;
  const attackerProfit = grossVaultOverpayment - manipulationCost;
```
