# Based on deployed contracts

Project: PM-01 — name-redacted prediction-market lending protocol
Category: Prediction-market collateral lending, leveraged outcome positions and asynchronous lending vault
Severity: Unrated — no standalone severity assigned in the source assessment; signer-dependent mechanism
Engagement: Independent hunting + responsible disclosure
Chain: Polygon Amoy testnet, chain ID 80002; source/fork snapshot block 51,383,934 (28 October 2026)
Tech stack: Solidity 0.8.26 / EVM, OpenZeppelin, ERC20, ERC1155, ERC4626 and ERC7540-style asynchronous redemption; Foundry / local Anvil fork tooling
Evidence: Historical local-fork test source and reported results; synthetic setup described below
Remediation: No confirmed remediation is established.
Contribution: Independent security research; program acceptance is not asserted.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Future-dated signed marks bypass the intended freshness horizon

## Description

The verifier enforces maximum age but not a maximum timestamp. A legitimately signed future mark passes now and remains valid until that future timestamp plus the configured window. A timestamp one year ahead therefore stretches a five-minute freshness policy by approximately one year. This requires a signer mistake/compromise or other availability of a valid future signature, not merely an untrusted caller choosing a new timestamp.

Despite its legacy test name, the accepted interval is finite. The recorded test checks acceptance at origination and a day later, not perpetual validity.

The deployment association is with the recorded Polygon Amoy testnet snapshot. Provider-runtime comparison does not establish independent recompilation, equivalence to an earlier audit revision, or a mainnet incident.

### Mechanism step by step

#### Step 1 — Sign a timestamp beyond current chain time

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

There is no timestamp<=now check. A future timestamp defeats the age comparison but does not bypass ECDSA or domain separation. An attacker cannot edit a signed timestamp without a new valid signature.

#### Step 2 — Originate immediately with the future high mark

Source: `src/BorrowerLogic.sol:40-51`

```solidity
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
```

Borrow binds the other quote and collateral fields, consumes a valid quote, verifies the future mark and sizes principal from its price. The PoC supplies both signatures with controlled test keys.

#### Step 3 — Accept a distinct future low mark a day later

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

The liquidation consumer uses the same missing upper-timestamp bound. The test constructs a different low-price signature for the Liquidator domain with the same future timestamp. It is not reuse of the identical high-price signature across domains.

### Impact and proof limits

A far-future timestamp can turn a signer-side error into a long-lived borrowing/liquidation input. Consequences still depend on the signed price and action eligibility. No standalone severity is assigned in `findings-after-source-2026-10-28.md:59`.

The receipt says reusedAfterSeconds:86400, but what persists is the future-timestamp condition: the test uses a high Pool-domain mark then a different low Liquidator-domain mark. The second action occurs at a one-day loan's term boundary; no incremental economic loss relative to ordinary expiry is measured. The future signature eventually expires at timestamp+freshness. The test name NeverStales is retained as an identifier, not endorsed as a mathematical claim.

### Remediation direction

Reject timestamps above current time plus a narrowly justified clock-skew allowance; preserve the independent maximum-age test. Check both domains at current, near-future, far-future and expired boundaries. Fixing this alone does not resolve V3 ordering.

No confirmed fix is established by the available evidence.

## Proof of concept

Recorded test `test_t6_futureDatedMarkNeverStales`, `poc/src/FinalProofs.t.sol:286-303`, part of the recorded six-test suite. The harness impersonates timelock to install controlled signers, mints lender/collateral balances and deliberately signs the future timestamps. No production signing-service incident is established.

## Relevant test excerpts

Controlled signer/funding setup:

Source: `poc/src/FinalProofs.t.sol:65-79`

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
        ctf.mint(borrower, TOKEN_ID, QTY);
        vm.prank(borrower);
        ctf.setApprovalForAll(POOL, true);
```

Future inputs, action and assertion:

Source: `poc/src/FinalProofs.t.sol:286-303`

```solidity
    function test_t6_futureDatedMarkNeverStales() public {
        uint64 t0 = uint64(block.timestamp);
        uint64 future = t0 + 365 days;
        // Borrow immediately with a mark dated one year ahead.
        PremiumQuote memory q = _quote(borrower, 6_000e6, 1 days, 1, t0 + 300);
        MarkAttestation memory mFuture = MarkAttestation(TOKEN_ID, MARK_060, future);
        bytes memory qs = _signQuote(q);
        bytes memory mfs = _signMark(pool.MARK_DOMAIN_SEPARATOR(), mFuture);
        vm.prank(borrower);
        uint256 loanId = pool.borrow(BorrowParams(TOKEN_ID, QTY, borrower), q, qs, mFuture, mfs);
        // A day later, a LOW mark with the same future date still passes freshness
        // and liquidates: staleness can never catch up to a future timestamp.
        MarkAttestation memory mLowFuture = MarkAttestation(TOKEN_ID, MARK_032, future);
        bytes memory slf = _signMark(liq.MARK_DOMAIN_SEPARATOR(), mLowFuture);
        vm.warp(t0 + 1 days);
        liq.triggerLiquidation(loanId, mLowFuture, slf);
        assertTrue(ledger.loan(loanId).status == LoanStatus.Liquidated, "future mark liquidates a day later");
        emit Receipt("N1_RESULT={futureMarkAccepted:true, reusedAfterSeconds:86400, stillFresh:true, rootCause:no timestamp<=block.timestamp check}");
```

Recorded receipt:

Source: `poc/README.md:53-53`

```text
N1_RESULT={futureMarkAccepted:true, reusedAfterSeconds:86400, stillFresh:true, rootCause:no timestamp<=block.timestamp check}
```
