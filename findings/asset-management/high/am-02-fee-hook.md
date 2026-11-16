# Based on deployed contracts

Project: Anonymous asset management project (AM-02)  
Category: Asset management  
Severity: High — report proposal; proxy scope, fee-as-yield classification and novelty require program ruling  
Engagement: Bug bounty hunting  
Chain: Ethereum; historical production-bytecode fork at block 26,243,324  
Tech stack: Solidity/EVM, fund proxies, fee manager, valuation/oracle registry, ERC-20, Anvil and Python JSON-RPC harness  
Evidence: Historical local or fork test excerpts; scope and fixture assumptions are described below  
Remediation: Not independently verified  
Contribution: Research and disclosure case study; exclusive discovery credit is not asserted  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Delisted tracked assets make ordinary redemptions skip accrued fund fees

## Description

A protocol governance delisting leaves a positive unsupported token among the fund’s tracked assets. PerformanceFee valuation then reverts inside the atomic fund-fee hook, rolling back earlier ManagementFee work. An ordinary shareholder can still redeem in kind because the Comptroller catches that failure and transfers tracked balances. The departed asset base can no longer support the skipped fee claim.

This is a manager-fee claim loss, not investor-principal theft or an immutable fund freeze. The owner can remove the unsupported tracked asset before redemption; governance can restore pricing. Those interventions can restore valuation but cannot recreate assets already redeemed. A diagnostic former-feed restoration measures 6,051.888130 USD of unrecovered value: 2,645.049772 management and 3,406.838358 performance. These dollar values are counterfactual local diagnostics, not current production prices. A historical cohort of 41 Ethereum funds is exposure context, not 41 measured losses or a current count. Prior audits documented related fail-open/valuation mechanisms; novelty remains a triage question.

### Mechanism step by step

#### Step 1 — Fail valuation for the positive unsupported tracked asset

With neither primitive nor derivative support, the valuation reverts. The production pin has the delisting precondition already present; the shareholder need not manufacture it.

Source: `contracts/release/infrastructure/value-interpreter/ValueInterpreter.sol:119-131`

```solidity
        // Handle case that asset is a primitive
        if (isSupportedPrimitiveAsset(_baseAsset)) {
            return __calcCanonicalValue(_baseAsset, _amount, _quoteAsset);
        }

        // Handle case that asset is a derivative
        address derivativePriceFeed = getPriceFeedForDerivative(_baseAsset);
        if (derivativePriceFeed != address(0)) {
            return __calcDerivativeValue(derivativePriceFeed, _baseAsset, _amount, _quoteAsset);
        }

        revert("__calcAssetValue: Unsupported _baseAsset");
    }
```

#### Step 2 — Roll back the combined fund-fee loop

A GAV-dependent fee calls valuation. Its revert unwinds the whole invokeHook call, including any earlier fee changes in the same call.

Source: `contracts/release/extensions/fee-manager/FeeManager.sol:280-292`

```solidity

        for (uint256 i; i < _fees.length; i++) {
            (bool settles, bool usesGav) = IFee(_fees[i]).settlesOnHook(_hook);
            if (!settles) {
                continue;
            }

            if (usesGav) {
                gav_ = __getGavAsNecessary(_comptrollerProxy, gav_);
            }

            __settleFee(_comptrollerProxy, _vaultProxy, _fees[i], _hook, _settlementData, gav_);
        }
```

#### Step 3 — Catch the fee failure without aborting redemption

The hook emits a failure event but does not revert the outer redemption.

Source: `contracts/release/core/fund/comptroller/ComptrollerLib.sol:913-928`

```solidity
    function __preRedeemSharesHook(
        address _redeemer,
        uint256 _sharesToRedeem,
        bool _forSpecifiedAssets,
        uint256 _gavIfCalculated
    ) private allowsPermissionedVaultAction {
        try IFeeManager(getFeeManager())
            .invokeHook(
                IFeeManager.FeeHook.PreRedeemShares,
                abi.encode(_redeemer, _sharesToRedeem, _forSpecifiedAssets),
                _gavIfCalculated
            ) {}
        catch (bytes memory reason) {
            emit PreRedeemSharesHookFailed(reason, _redeemer, _sharesToRedeem);
        }
    }
```

#### Step 4 — Pay tracked assets after the failed fund-fee hook

The ordinary redemption transfers pro-rata assets; the protocol fee is separate and still settles. No debt record preserves the skipped fund-fee amount against the departed assets.

Source: `contracts/release/core/fund/comptroller/ComptrollerLib.sol:818-827`

```solidity
        payoutAmounts_ = new uint256[](payoutAssets_.length);
        for (uint256 i; i < payoutAssets_.length; i++) {
            payoutAmounts_[i] = ERC20(payoutAssets_[i]).balanceOf(vaultProxy).mul(sharesToRedeem).div(sharesSupply);

            // Transfer payout asset to _recipient
            if (payoutAmounts_[i] > 0) {
                IVault(vaultProxy).withdrawAssetTo(payoutAssets_[i], _recipient, payoutAmounts_[i]);
            }
        }

```

## Proof of concept

The Python harness forks unmodified production bytecode and impersonates an actual shareholder locally, funding ETH for transaction execution. Owner accounts are also available for controls. In the production-state arm, the in-kind redemption succeeds, a hook-failure event is emitted and no fund fee settles. Continuous settlement fails before and after exit. A separate diagnostic arm temporarily restores the former price feed under owner authority, measures the fee claim, then reverts its snapshot. It estimates 6,161.218741 USD before exit and 109.330611 residual recovery, a 6,051.888130 difference. The diagnostic pricing and owner intervention are not permissions available to the shareholder.

**Proof limits:** Historical state and counterfactual restored-feed valuation; program scope and novelty are unresolved.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `report-preparation/[project]/poc_[project]_fee_skip.py:300-332`

```python
        print("STEP 2 — control: permissionless fee settlement reverts (fees uncollectible in current state)")

        # STEP 3 (main arm). The holder redeems its full position in-kind.
        # redeemSharesInKind (ComptrollerLib.sol:783-831) pays each tracked
        # token pro-rata from raw vault balances. Its pre-redemption hook
        # (ComptrollerLib.sol:913-928) wraps the FeeManager call in try/catch,
        # so when fee settlement reverts the redemption continues anyway.
        # The receipt must show exactly one PreRedeemSharesHookFailed event
        # and zero fee settlement events.
        holder_before = {a: balance_of(a, HOLDER) for a in TRACKED}
        receipt = transact(HOLDER, COMPTROLLER, calldata_redeem_in_kind(HOLDER))
        if int(receipt["status"], 16) != 1:
            fail("in-kind redemption reverted; expected it to complete")

        hook_failures, fee_events, payouts = [], [], {}
        for entry in receipt["logs"]:
            topic0 = entry["topics"][0].lower()
            if topic0 == TOPIC_HOOK_FAILED:
                hook_failures.append(entry)
            elif topic0 in (TOPIC_FEE_SETTLED, TOPIC_FEE_CONTRACT_SETTLED):
                fee_events.append(entry)
            elif topic0 == TOPIC_TRANSFER and as_address(entry["topics"][1]) == VAULT \
                    and as_address(entry["topics"][2]) == HOLDER:
                payouts[entry["address"].lower()] = int(entry["data"], 16)

        if len(hook_failures) != 1:
            fail(f"expected exactly one PreRedeemSharesHookFailed event, got {len(hook_failures)}")
        failure_hash = hook_failures[0]["topics"][1].lower()
        if failure_hash != EXPECTED_HOOK_FAILURE_HASH:
            fail(f"hook failure reason hash {failure_hash} does not match the recorded MKR pricing failure")
        # The protocol fee settlement is a separate call path and does fire;
        # the fund's own Management/Performance fees must not.
        print("STEP 3 — main arm: in-kind redemption completed")
```

Source: `report-preparation/[project]/poc_[project]_fee_skip.py:369-382`

```python
            restore(root)  # back to untouched pin state

            def settle_and_measure(tag):
                snap = snapshot()
                try:
                    tx = transact(PROTOCOL_OWNER, VALUE_INTERPRETER, calldata_restore_mkr_feed())
                    if int(tx["status"], 16) != 1:
                        fail(f"{tag}: diagnostic feed registration failed")
                    ok, gav_raw = try_call(COMPTROLLER, SEL["calcGav"])
                    if not ok:
                        fail(f"{tag}: calcGav still reverts after diagnostic override")
                    gav = int(gav_raw, 16)
                    pre_supply = int(try_call(VAULT, SEL["totalSupply"])[1], 16)
                    tx = transact(HOLDER, COMPTROLLER, calldata_continuous_settlement())
```
