# Source-code-based

Project: PM-01 — name-redacted prediction-market lending protocol
Category: Prediction-market collateral lending, leveraged outcome positions and asynchronous lending vault
Severity: Low — conditional production-ramp hardening; current fixed-rate slippage claim downgraded
Engagement: Independent hunting + responsible disclosure
Chain: Polygon Amoy testnet, chain ID 80002; source/fork snapshot block 51,383,934 (28 October 2026)
Tech stack: Solidity 0.8.26 / EVM, OpenZeppelin, ERC20, ERC1155, ERC4626 and ERC7540-style asynchronous redemption; Foundry / local Anvil fork tooling
Evidence: Historical source and configuration
Remediation: No confirmed remediation is established.
Contribution: Independent security research; program acceptance is not asserted.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Zap conversion lacks a caller minimum-output bound

## Description

Zap accepts input assets or shares without a caller minimum output. It correctly measures real ramp balance changes and passes those amounts through. If a future/production integration is price-taking, however, measuring a bad execution does not make it acceptable to the caller.

The conditional Low rating reflects the testnet ramps’ fixed 1:1 minting. No extractable current slippage or production variable-price configuration is established. Ramp references are immutable in this Zap (`src/periphery/Zap.sol:25-32`).

The deployment association is with the recorded Polygon Amoy testnet snapshot. Provider-runtime comparison does not establish independent recompilation, equivalence to an earlier audit revision, or a mainnet incident.

### Mechanism step by step

#### Step 1 — Convert and deposit without a minimum-share comparison

Source: `src/periphery/Zap.sol:55-69`

```solidity
    function depositWith(address token, uint256 amount, address receiver) external returns (uint256 shares) {
        if (!allowedEntryAsset[token]) revert Errors.EntryAssetNotAllowed(token);

        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
        IERC20(token).forceApprove(onramp, amount);

        // What the ramp minted, measured rather than assumed: USDC.e converts 1:1 today, but the
        // deposit must carry the true amount for any listed asset on any day.
        uint256 balanceBefore = pusd.balanceOf(address(this));
        IOnramp(onramp).wrap(token, address(this), amount);
        uint256 minted = pusd.balanceOf(address(this)) - balanceBefore;

        pusd.forceApprove(vault, minted);
        shares = IERC4626(vault).deposit(minted, receiver);
    }
```

The allowlist controls eligibility, while actual pUSD delta controls the vault deposit. There is no minSharesOut parameter. This is an execution-intent protection gap, not nominal-versus-actual balance misaccounting.

#### Step 2 — Unwrap without a minimum-exit-asset comparison

Source: `src/periphery/Zap.sol:72-81`

```solidity
    function claimAndUnwrap(uint256 shares, address receiver) external returns (uint256 amountOut) {
        // The vault enforces the operator gate: this call stands only if msg.sender granted the
        // Zap operator status (ERC-7540). Assets land here, already the caller's by fulfillment.
        uint256 assets = IERC4626(vault).redeem(shares, address(this), msg.sender);

        pusd.forceApprove(offramp, assets);
        uint256 balanceBefore = IERC20(exitAsset).balanceOf(receiver);
        IOfframp(offramp).unwrap(exitAsset, receiver, assets);
        amountOut = IERC20(exitAsset).balanceOf(receiver) - balanceBefore;
    }
```

Claimable shares and user-granted Zap operator authority are prerequisites. The exit reports the receiver’s real delta but does not compare it with a user floor.

#### Step 3 — Recognize that testnet ramp output is fixed

Source: `src/test/mocks/MockVenueRamps.sol:39-43`

```solidity
    function wrap(address _asset, address _to, uint256 _amount) external {
        if (paused) revert OnlyUnpaused();
        IERC20(_asset).transferFrom(msg.sender, address(this), _amount);
        pusd.mint(_to, _amount);
    }
```

The onramp pulls amount and mints that same amount. The offramp likewise mints 1:1 (`src/test/mocks/MockVenueRamps.sol:64-68`). A pause causes an atomic revert, not proven value loss. A variable-price integration is required to activate the slippage theory.

### Impact and proof limits

The supported issue is conditional absence of caller outcome bounds. Balance-delta accounting and token allowlisting are real defenses. No measured loss, profitable sandwich, passing slippage exploit or live variable-price ramp is established.

### Remediation direction

For variable-price production integrations, expose minimum deposit shares and minimum exit assets and revert when actual output is below them. Preserve delta accounting and document any fixed-rate assumption.

No confirmed fix is established by the available evidence.

## Proof of concept

No dedicated exploit test or demonstrated variable-price ramp loss is available. The evidence is the missing caller output bound and the fixed 1:1 testnet integration. A variable-price integration would need a separate executable proof.

## Relevant test excerpts

No finding-specific executable test excerpt is available. The source excerpts above are the supporting evidence.
