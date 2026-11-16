# Based on deployed contracts

Project: PM-01 — name-redacted prediction-market lending protocol
Category: Prediction-market collateral lending, leveraged outcome positions and asynchronous lending vault
Severity: Low — voluntary liquidity/UX constraint; operator-lock hypothesis refuted
Engagement: Independent hunting + responsible disclosure
Chain: Polygon Amoy testnet, chain ID 80002; source/fork snapshot block 51,383,934 (28 October 2026)
Tech stack: Solidity 0.8.26 / EVM, OpenZeppelin, ERC20, ERC1155, ERC4626 and ERC7540-style asynchronous redemption; Foundry / local Anvil fork tooling
Evidence: Historical local-fork test source and reported results; synthetic setup described below
Remediation: No confirmed remediation is established.
Contribution: Independent security research; program acceptance is not asserted.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Queued redemption cannot be canceled while awaiting liquidity

## Description

A lender entering the asynchronous redemption queue cannot cancel/reclaim its still-pending request through the affected API. It may therefore be unable to reverse an exit decision during a cash shortfall. Pending shares nevertheless retain income, loss and recovery exposure until fulfillment.

The queue is self-only: requestRedeem cannot queue another owner’s shares. Deposits automatically crank, and anyone can crank manually; the remaining constraint is cash availability, not exclusive keeper authorization.

The deployment association is with the recorded Polygon Amoy testnet snapshot. Provider-runtime comparison does not establish independent recompilation, equivalence to an earlier audit revision, or a mainnet incident.

### Mechanism step by step

#### Step 1 — Require the owner to submit the request

Source: `src/LatUSD.sol:225-247`

```solidity
    function requestRedeem(uint256 shares, address controller, address owner)
        external
        nonReentrant
        returns (uint256 requestId)
    {
        if (shares < minRequest) revert Errors.BelowMinRequest(shares, minRequest);
        if (controller == address(0)) revert Errors.ZeroAddress();
        // Self-controlled requests only in v1: no operator surface to audit.
        if (msg.sender != owner) revert Errors.Unauthorized(msg.sender);

        // Return value deliberately unused: the call is the accrue-then-act sync, not a read.
        // slither-disable-next-line unused-return
        ledger.accrue();

        requestId = _requests.length;
        bool nothingAhead = _head == requestId;

        // Escrow: shares transfer in and stay ordinary shares — earning income, bearing trigger
        // losses — until they burn at fulfillment (D3, I19).
        _transfer(owner, address(this), shares);
        _requests.push(Request({controller: controller, shares: shares, fulfilled: 0}));
        _pendingShares[controller] += shares;
        queuedShares += shares;
```

An operator cannot force this request for the owner. Shares move to escrow and pending balances are recorded; no cancellation transition appears in the affected API.

#### Step 2 — Fulfill immediately when cash permits

Source: `src/LatUSD.sol:254-267`

```solidity
        // Empty queue ahead and idle covers the whole request: fulfill inside this transaction,
        // so the calm-market experience is indistinguishable from synchronous redeem.
        if (nothingAhead && _convertToAssets(shares, Math.Rounding.Ceil) <= ledger.idle()) {
            _crank(1);
        }
    }

    /// @inheritdoc ILatUSD
    function crank(uint256 maxN) external nonReentrant returns (uint256 requestsTouched, uint256 sharesFulfilled) {
        // Return value deliberately unused: the call is the accrue-then-act sync, not a read.
        // slither-disable-next-line unused-return
        ledger.accrue();
        (requestsTouched, sharesFulfilled) = _crank(maxN);
    }
```

A clear queue and enough idle can fulfill synchronously; otherwise permissionless crank progresses FIFO. Deposits/mints also invoke automatic bounded crank (`src/LatUSD.sol:181-210`).

#### Step 3 — Stop at real liquidity exhaustion

Source: `src/LatUSD.sol:293-298`

```solidity
                // What idle can cover at this moment's NAV; both roundings favor the pool, and the
                // down-down round trip guarantees fillAssets never exceeds idle.
                uint256 coverable = _convertToShares(ledger.idle(), Math.Rounding.Floor);
                fillShares = remaining <= coverable ? remaining : coverable;
                fillAssets = _convertToAssets(fillShares, Math.Rounding.Floor);
                if (fillAssets == 0) break; // idle exhausted: the line holds
```

Processing converts only what idle covers. The user remains queued if meaningful output cannot be funded, without an API to reverse that pending request.

#### Step 4 — Preserve pending economic ownership

Source: `src/LatUSD.sol:523-533`

```solidity
    function _checkpointEconomic(address account, uint48 epoch) internal {
        if (account == address(0) || account == address(this)) return;
        uint256 shares = balanceOf(account) + _pendingShares[account];
        // slither-disable-next-line unused-return
        _economicCheckpoints[account].push(epoch, SafeCast.toUint208(shares));
    }

    function _checkpointSupply(uint48 epoch) internal {
        uint256 shares = totalSupply() - balanceOf(address(this)) + queuedShares;
        // slither-disable-next-line unused-return
        _supplyCheckpoints.push(epoch, SafeCast.toUint208(shares));
```

Queued shares stay in the controller’s economic balance/supply. This is not confiscation of all lender rights; once cash is assigned, normal claim methods apply.

### Impact and proof limits

The supported impact is inability to cancel a voluntary request during insufficient liquidity. Theft, unauthorized operator locking, permanent freeze and exclusive keeper control are not established. Adding cancellation needs economic review of seniority and N2 interactions.

### Remediation direction

Explain request irreversibility before submission. If cancellation is desired, specify treatment of partial fulfillment, queue liability and seniority, while retaining self-only requests and pending economic attribution.

No confirmed fix is established by the available evidence.

## Proof of concept

The queue tests for N2 show an owner’s shares entering the queue and retaining recovery exposure. They are supporting evidence, not a dedicated test of cancellation or a separate V6 profit. Setup uses synthetic deposits, transferred shares and a privileged modeled remittance.

## Relevant test excerpts

Supporting N2 owner action and assertion:

Source: `poc/src/N2QueueRoll.t.sol:89-94`

```solidity
        uint256 allWhaleShares = whaleShares + smallShares;
        vm.prank(whale);
        vault.requestRedeem(allWhaleShares, whale, whale);
        uint256 queueLiability = vault.queueLiability();
        uint256 idleAfterQueue = ledger.idle();
        assertTrue(queueLiability > idleAfterQueue, "queue must outrank idle");
```

Source: `poc/src/N2QueueRoll.t.sol:120-123`

```solidity
        // Empirical crux: every share is escrowed in the pending request, yet
        // the loss-epoch snapshot still assigns all of them to the controller.
        uint256 economicAtLoss = vault.economicBalanceAt(whale, 1);
        assertEq(economicAtLoss, allWhaleShares, "pending shares counted at loss");
```

Supporting observations, not a V6-specific receipt:

Source: `poc/N2-OBSERVED.md:5-9`

```text
5. Queue gate at roll: liability 500054258185 > idle 494000000000.
6. Fresh valid roll attempts reverted with RollBlockedByQueue: 2 of 2.
7. Expiry claim status: Expired; recorded pool loss was 5993967122.
8. Sale remittance: 12000000000; whale recovery claim paid 10793967122.
9. Redemption payout: 494060328767; total whale recovered: 504854295889.
```
