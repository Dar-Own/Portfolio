# Based on deployed contracts

Project: PM-01 — name-redacted prediction-market lending protocol
Category: Prediction-market collateral lending, leveraged outcome positions and asynchronous lending vault
Severity: Informational — bounded residual; zero-supply scenario structurally refuted
Engagement: Independent hunting + responsible disclosure
Chain: Polygon Amoy testnet, chain ID 80002; source/fork snapshot block 51,383,934 (28 October 2026)
Tech stack: Solidity 0.8.26 / EVM, OpenZeppelin, ERC20, ERC1155, ERC4626 and ERC7540-style asynchronous redemption; Foundry / local Anvil fork tooling
Evidence: Historical source and configuration
Remediation: No confirmed remediation is established.
Contribution: Independent security research; program acceptance is not asserted.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Pro-rata recovery rounding leaves protected reserved dust

## Description

Individual recovery entitlements round down. Their sum can leave a small part of cumulative credited recovery unclaimable; claims reduce recoveryReserved only by paid amounts, while sweep excludes that reserve. The residual remains protected rather than enriching a later depositor.

The observed effect is rounding dust; the zero-supply variant is structurally unreachable under the cash/principal accounting. No material accumulation or attacker extraction is demonstrated.

The deployment association is with the recorded Polygon Amoy testnet snapshot. Provider-runtime comparison does not establish independent recompilation, equivalence to an earlier audit revision, or a mainnet incident.

### Mechanism step by step

#### Step 1 — Reserve the full recovery credit

Source: `src/LatUSD.sol:442-452`

```solidity
    function creditRecovery(uint256 id, uint256 amount) external nonReentrant {
        if (!ledger.authorized(msg.sender)) revert Errors.Unauthorized(msg.sender);
        uint256 epoch = ledger.lossEpochOf(id);
        if (epoch == 0) revert Errors.NoLossEpoch(id);

        emit Events.RecoveryCredited(id, amount, epoch);
        // Always reserved, never NAV: pricing a recovery into the next entry is exactly the
        // capture ADR-7 removes (Zenith #9, #16). If nobody held at the loss, the credit is
        // simply claimable by no one.
        recoveryCredited[id] += amount;
        recoveryReserved += amount;
```

Authorized credit increments both per-record cumulative recovery and aggregate reserve. The reserve is outside ordinary NAV.

#### Step 2 — Floor each holder’s cumulative entitlement

Source: `src/LatUSD.sol:480-488`

```solidity
    function recoveryClaimable(uint256 id, address controller) public view returns (uint256 assets) {
        uint256 epoch = ledger.lossEpochOf(id);
        if (epoch == 0) return 0;
        uint256 supply = economicSupplyAt(epoch);
        if (supply == 0) return 0;

        uint256 entitled = MathLib.mulDivDown(economicBalanceAt(controller, epoch), recoveryCredited[id], supply);
        uint256 claimed = _recoveryClaimed[id][controller];
        assets = entitled > claimed ? entitled - claimed : 0;
```

Entitlement is floor(credit×historicalBalance/historicalSupply), less prior claims. Each holder’s current cumulative floor error is less than one asset base unit. Later credits can absorb interim rounding for that record.

#### Step 3 — Subtract only actual paid claims

Source: `src/LatUSD.sol:466-476`

```solidity
        for (uint256 i = 0; i < ids.length; ++i) {
            uint256 owed = recoveryClaimable(ids[i], controller);
            if (owed == 0) continue;
            _recoveryClaimed[ids[i]][controller] += owed;
            assets += owed;
            emit Events.RecoveryClaimed(controller, receiver, ids[i], owed);
        }
        if (assets == 0) revert Errors.NothingToClaim();

        recoveryReserved -= assets;
        IERC20(asset()).safeTransfer(receiver, assets);
```

Amounts no holder is entitled to never flow through this subtraction. Repeated claims cannot exceed cumulative individual entitlement.

#### Step 4 — Protect the remaining reserve from sweeping

Source: `src/LatUSD.sol:425-434`

```solidity
        // the lenders who held at their losses, and this vault's own shares held in queue escrow.
        uint256 reserved = 0;
        if (token == asset()) {
            reserved = ledger.idle() + assignedAssets + ledger.streamReservoir() + recoveryReserved;
        } else if (token == address(this)) {
            reserved = queuedShares;
        }

        uint256 excess = IERC20(token).balanceOf(address(this)) - reserved;
        IERC20(token).safeTransfer(to, excess);
```

Sweep excludes recoveryReserved. The zero-supply guard exists, but existence of that branch is not evidence of a reachable zero-holder loss. Snapshot lookup retains prior economic supply (`src/LatUSD.sol:491-501`); the accounting reachability analysis says outstanding principal and required exit cash prevent the alleged state.

### Impact and proof limits

One base unit of the six-decimal asset is 0.000001 pUSD. The bound is less than one unit per holder of a cumulative allocation, not an arbitrary percentage. Later cumulative credits can make earlier fractional entitlements payable, so each partial remittance does not create a permanent independent rounding loss. Many separate records can accumulate dust, but a material deployed amount, permissionless attack or zero-supply reproduction is not established.

### Remediation direction

Document the rounding bound. If reclamation is desired, specify a safe finalization/claim-expiry or remainder-distribution policy without exposing valid unpaid claims. Require a reachable measured accumulation before increasing severity.

No confirmed fix is established by the available evidence.

## Proof of concept

No dedicated exploit test, material dust-accumulation measurement or zero-supply reproduction is available. The evidence is integer rounding and reserve accounting; extraction of value is not established.

## Relevant test excerpts

No finding-specific executable test excerpt is available. The source excerpts above are the supporting evidence.
