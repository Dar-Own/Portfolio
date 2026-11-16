# Based on deployed contracts

Project: PM-01 — name-redacted prediction-market lending protocol
Category: Prediction-market collateral lending, leveraged outcome positions and asynchronous lending vault
Severity: Medium — liveness and privileged operational dependency
Engagement: Independent hunting + responsible disclosure
Chain: Polygon Amoy testnet, chain ID 80002; source/fork snapshot block 51,383,934 (28 October 2026)
Tech stack: Solidity 0.8.26 / EVM, OpenZeppelin, ERC20, ERC1155, ERC4626 and ERC7540-style asynchronous redemption; Foundry / local Anvil fork tooling
Evidence: Historical local-fork test source and reported results; synthetic setup described below
Remediation: No confirmed remediation is established.
Contribution: Independent security research; program acceptance is not asserted.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Executor outage blocks leverage-position exits

## Description

Every affected leverage-position exit or rollover path depends on one executor address. A guardian can revoke that address immediately, setting it to zero; a lost key or unavailable executor produces the same lack of an authorized caller. Traders cannot close, reduce a position, remove margin or roll the term themselves. The permissionless timeout path handles reservations, not already finalized positions.

The fork proof opens a position, revokes the executor through the configured guardian, checks four gated calls revert, and observes that the position remains open after its epoch. Adding margin still works. It then restores the executor by impersonating the timelock, funds that executor, and closes the position. This establishes an exit-liveness dependency and an authorized recovery route. It does not establish an automatic timeout or a 300-second maximum freeze.

The deployment census records the same controller as guardian and executor. That explains why the PoC's `vm.prank(EXECUTOR)` can invoke guardian revocation; it is not a generally permissionless revocation method.

The deployment association is with the recorded Polygon Amoy testnet snapshot. Provider-runtime comparison does not establish independent recompilation, equivalence to an earlier audit revision, or a mainnet incident.

### Mechanism step by step

#### Step 1 — Finalize a funded leverage position

Source: `src/Ledger.sol:1236-1254`

```solidity
            posId = intentId;
            uint64 start = uint64(block.timestamp);
            uint64 end = start + r.epoch;
            _positions[posId] = Position({
                tokenId: r.tokenId,
                trader: r.trader,
                epochEnd: end,
                status: PositionStatus.Open,
                notional: filled.toUint128(),
                shares: r.filledShares,
                margin: marginKept.toUint128(),
                premium: premium.toUint128(),
                maxChargeBps: r.maxChargeBps,
                maxTotalCharges: r.maxTotalCharges,
                chargesPaid: charges.toUint128()
            });

            // The lender share streams over the position's term, exactly as a loan's (I23).
            _startStream(posId, lenderShare, start, end);
```

An authorized executor opens and finalizes a funded leverage position using a valid quote and self-reported fill (`src/Leverage.sol:111-176`; `src/Ledger.sol:1236-1254`). The test supplies controlled quote signing and trader/lender funds.

#### Step 2 — Revoke the only authorized caller

Source: `src/Leverage.sol:104-108`

```solidity
    function revokeExecutor() external {
        if (msg.sender != guardian) revert Errors.Unauthorized(msg.sender);
        emit Events.ExecutorSet(executor, address(0));
        executor = address(0);
    }
```

The guardian calls `revokeExecutor`, replacing the executor with zero. Only the timelock can install a nonzero replacement (`src/Leverage.sol:82-92,104-107`). If the existing executor merely stops serving, it remains configured but unavailable; the practical caller dependency remains.

#### Step 3 — Reject user and former-executor exits

Source: `src/Leverage.sol:82-85`

```solidity
    modifier onlyExecutor() {
        if (msg.sender != executor) revert Errors.Unauthorized(msg.sender);
        _;
    }
```

`closeLeverage`, `removeFromPosition`, `removeMargin` and `rollLeverage` all require `onlyExecutor`. With a revoked address, neither the former executor nor the trader can pass that gate (`src/Leverage.sol:226,266-269,332-335,372`).

#### Step 4 — Reject reservation timeout for finalized exposure

Source: `src/Ledger.sol:697-708`

```solidity
        Reservation storage r = _reservations[intentId];
        if (r.status != ReservationStatus.Reserved) revert Errors.ReservationNotActive(intentId);
        uint64 deadline = r.ttl + timeoutGrace;
        if (uint64(block.timestamp) <= deadline) revert Errors.TimeoutNotReached(intentId, deadline);

        // What the placing wallet owed the vault: everything at zero fills, the unfilled remainder
        // past the first fill. The gap to what actually came back is the executor's debt — never
        // the trader's, so the refund legs are exactly the voluntary paths'.
        uint256 owed;
        if (r.filledNotional == 0) {
            owed = r.principal;
            (, refund) = _abort(intentId, principalReturned);
```

`timeoutAbort` reads a reservation and calls `timeoutReservation`, which requires `ReservationStatus.Reserved`. Completion has already changed that status to `Finalized`; the timeout is not an emergency close for the associated open position (`src/Leverage.sol:194-207`; `src/Ledger.sol:697-708,1215`).

#### Step 5 — End income streaming without automatic close

Source: `src/Ledger.sol:1414-1424`

```solidity
        while (cursor < limit) {
            uint256 next = _nextEnd(cursor + 1, limit);
            if (next == 0) break;
            if (walked == maxEnds) return (indexRay, rateRay, cursor.toUint64());
            indexRay += MathLib.mulDivDown(rateRay, next - cursor, 1);
            rateRay -= _endingRateRay[next.toUint64()];
            cursor = next;
            ++walked;
        }
        indexRay += MathLib.mulDivDown(rateRay, to - cursor, 1);
        syncedTo = to;
```

Passing `epochEnd` does not automatically close a position or trigger the direct-loan liquidation path. The position's prepaid income stream ends at its registered end while principal can remain outstanding (`src/Ledger.sol:1236-1254,1397-1424`). The resulting unpriced extension is source-supported; the L1 test does not measure a numerical lender-yield loss or simulate an underwater venue position.

#### Step 6 — Require privileged executor restoration

Source: `src/Leverage.sol:88-93`

```solidity
    function setExecutor(address newExecutor) external {
        if (msg.sender != timelock) revert Errors.Unauthorized(msg.sender);
        if (newExecutor == address(0)) revert Errors.ZeroAddress();
        emit Events.ExecutorSet(executor, newExecutor);
        executor = newExecutor;
    }
```

Governance must restore a usable executor, and that executor must possess/return the required funds to close. The test demonstrates this with direct timelock impersonation and newly minted executor funds. The configured 300-second delay is a minimum scheduling interval for a fresh authorized operation, not a maximum operational recovery time (`poc/src/FinalProofs.t.sol:274-281`; `src/lib/openzeppelin-contracts/contracts/governance/TimelockController.sol:314-322`).

### Impact and proof limits

An executor outage or revocation can strand open leverage positions and disable user-initiated risk reduction. The recorded proof checks four gated methods, leaves the position open beyond a one-hour epoch, and succeeds at close after administrative restoration. The caller restriction is confirmed in source; the test uses generic `expectRevert()` rather than matching the precise `Unauthorized` error, and its roll arguments are reused setup data. The receipt's four blocked calls should therefore be read together with source, not as four independently isolated economic exploits.

The title refers to leverage positions. Direct loans have separate liquidation/expiry handling; this finding does not establish that all protocol exits or lender redemptions are executor-gated. `addMargin` remains available, and reservation timeout remains available for qualifying reservations. Neither returns an already finalized position's assets. No permissionless forced close or liquidation for leverage positions is established in the affected implementation.

Recovery is operationally contingent: someone must retain proposal/execution authority, install a working executor and arrange venue/cash settlement. If the controller also holds concentrated governance authority and loses the relevant key, the available evidence does not establish a bounded recovery route. Medium reflects the source's liveness assessment, not a claim of permanent fund loss or a production outage.

## Proof of concept

**Recorded test:** `test_t5_executorRevocationFreezesExits`, `poc/src/FinalProofs.t.sol:235-282`.

Setup creates a local fork at the pinned Amoy block, installs controlled mark/quote signers by impersonating the timelock, mints a 100,000 pUSD lender deposit and funds a trader with 12,000 pUSD. The test opens 40,000 notional with 10,000 margin and a controlled quote. Mock/test token minting and `vm.prank` provide the actors and funds; none is evidence of public access to privileged production roles.

## Relevant test excerpts

Signer control and lender funding, with the preceding RPC-bearing fork line omitted:

Source: `poc/src/FinalProofs.t.sol:65-76`

```solidity
        markSigner = vm.addr(markKey);
        quoteSigner = vm.addr(quoteKey);
        vm.startPrank(TL);
        pool.setMarkSigner(markSigner);
        liq.setMarkSigner(markSigner);
        oracle.setSigner(quoteSigner);
        vm.stopPrank();
        pusd.mint(lender, 100_000e6);
        vm.startPrank(lender);
        pusd.approve(VAULT, type(uint256).max);
        vault.deposit(100_000e6, lender);
        vm.stopPrank();
```

Position setup and authorized opening:

Source: `poc/src/FinalProofs.t.sol:237-250`

```solidity
        address trader = address(0x4444);
        uint256 principal = 40_000e6;
        pusd.mint(trader, 12_000e6);
        vm.prank(trader);
        pusd.approve(LEVERAGE, type(uint256).max);
        LeverageQuote memory q = LeverageQuote(bytes32("t5"), trader, TOKEN_ID, principal, 3600, 100, 11, uint64(block.timestamp) + 300);
        OpenIntent memory i = OpenIntent(bytes32("t5"), trader, TOKEN_ID, 10_000e6, principal, uint64(block.timestamp) + 600, 0, 0);
        bytes memory lqSig = _signLq(q);
        vm.startPrank(EXECUTOR);
        uint256 intentId = lev.reserveOpen(i, q, lqSig);
        lev.recordFill(intentId, FillRef(bytes32("f1"), principal, 66_666e6));
        lev.completeOpen(intentId);
        vm.stopPrank();
        uint256 posId = intentId;
```

Guardian revocation followed by the four rejected executor calls:

Source: `poc/src/FinalProofs.t.sol:252-263`

```solidity
        // Guardian revokes the executor (same EOA on this deployment).
        vm.prank(EXECUTOR);
        lev.revokeExecutor();

        LeverageQuote memory dummy = q;
        bytes memory dummySig = lqSig;
        uint256 blocked;
        vm.expectRevert(); vm.prank(EXECUTOR); lev.closeLeverage(posId, principal); blocked++;
        vm.expectRevert(); vm.prank(EXECUTOR); lev.removeFromPosition(posId, 1e6, 0); blocked++;
        vm.expectRevert(); vm.prank(EXECUTOR); lev.removeMargin(posId, 1e6); blocked++;
        vm.expectRevert(); vm.prank(EXECUTOR); lev.rollLeverage(posId, dummy, dummySig); blocked++;
        assertEq(blocked, 4, "all four executor-gated exits frozen");
```

The positive control preserves permissionless margin-in. The position remains open after expiry; privileged, funded recovery closes it:

Source: `poc/src/FinalProofs.t.sol:265-281`

```solidity
        // The one permissionless adjustment still works (margin in is always safe).
        pusd.mint(trader, 1_000e6);
        vm.prank(trader);
        lev.addMargin(posId, 1_000e6);

        // Past epochEnd the position still sits open; only governance restores exits.
        vm.warp(block.timestamp + 3601);
        Position memory p = ledger.position(posId);
        assertTrue(p.status == PositionStatus.Open, "position stranded past epoch end");
        vm.prank(TL);
        lev.setExecutor(EXECUTOR);
        pusd.mint(EXECUTOR, principal);
        vm.startPrank(EXECUTOR);
        pusd.approve(LEVERAGE, principal);
        lev.closeLeverage(posId, principal);
        vm.stopPrank();
        assertTrue(ledger.position(posId).status == PositionStatus.Closed, "timelock restore unfreezes");
```

Recorded receipt:

Source: `poc/README.md:52-52`

```text
L1_RESULT={exitsBlocked:4, marginInStillOpen:true, strandedPastEpochEnd:true, restoredByTimelock:true, freezeBound:minDelay(300s)}
```

`freezeBound:minDelay(300s)` is retained verbatim from the receipt but is not a proven upper bound. The restoration code neither schedules nor executes through the timelock's operation queue; it directly impersonates the timelock. The evidence proves the authorization boundary and successful restored-executor close, not the elapsed time or independent availability of real governance.

### Remediation direction

Preserve an independently controlled path for safe position reduction and close during executor outages. If venue settlement prevents a fully permissionless close, define and enforce a redundant executor/recovery process with bounded obligations and clear user-visible outage semantics. Separate emergency revocation from the only recovery authority. Specify how overdue leverage positions accrue charges or must settle after `epochEnd`, and test finalized-position recovery independently from reservation timeout.

No implemented remediation or post-fix validation is established by the available evidence.
