# Based on deployed contracts

Project: PM-01 — name-redacted prediction-market lending protocol
Category: Prediction-market collateral lending, leveraged outcome positions and asynchronous lending vault
Severity: Medium — governance/mainnet posture; authorized configuration risk
Engagement: Independent hunting + responsible disclosure
Chain: Polygon Amoy testnet, chain ID 80002; source/fork snapshot block 51,383,934 (28 October 2026)
Tech stack: Solidity 0.8.26 / EVM, OpenZeppelin, ERC20, ERC1155, ERC4626 and ERC7540-style asynchronous redemption; Foundry / local Anvil fork tooling
Evidence: Historical source and configuration
Remediation: No confirmed remediation is established.
Contribution: Independent security research; program acceptance is not asserted.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Five-minute governance delay for critical policy changes

## Description

The recorded Amoy deployment uses a 300-second minimum timelock delay for changes to executor and signer authority, mark freshness, liquidation treasury/surplus policy and breaker state. The source-review register also records one controller holding proposer, executor and canceller roles. A hostile or erroneous authorized proposal can therefore become executable five minutes after scheduling, without a second independent key being required by those roles.

OpenZeppelin's timelock enforces the configured delay correctly. The issue is the chosen response window for this authority, especially when lender exits are asynchronous and depend on cash. Five minutes of proposal visibility does not guarantee that a lender can finish redeeming before a policy change. The earlier census is pinned to block 51,203,794; the later 28 October source-review register confirms the 300-second posture and role concentration. These are recorded observations, not a fresh chain read.

The deployment association is with the recorded Polygon Amoy testnet snapshot. Provider-runtime comparison does not establish independent recompilation, equivalence to an earlier audit revision, or a mainnet incident.

### Mechanism step by step

#### Step 1 — Schedule at the configured minimum

Source: `src/lib/openzeppelin-contracts/contracts/governance/TimelockController.sol:314-323`

```solidity
    function _schedule(bytes32 id, uint256 delay) private {
        if (isOperation(id)) {
            revert TimelockUnexpectedOperationState(id, _encodeStateBitmap(OperationState.Unset));
        }
        uint256 minDelay = getMinDelay();
        if (delay < minDelay) {
            revert TimelockInsufficientDelay(delay, minDelay);
        }
        _timestamps[id] = block.timestamp + delay;
    }
```

A caller holding `PROPOSER_ROLE` schedules a target call or batch with delay 300 seconds. `_schedule` checks only the configured minimum; it does not impose a longer risk-based delay for signer, custody or withdrawal-sensitive changes (`src/lib/openzeppelin-contracts/contracts/governance/TimelockController.sol:264-278,314-322`).

#### Step 2 — Execute the ready operation

Source: `src/lib/openzeppelin-contracts/contracts/governance/TimelockController.sol:356-369`

```solidity
    function execute(
        address target,
        uint256 value,
        bytes calldata payload,
        bytes32 predecessor,
        bytes32 salt
    ) public payable virtual onlyRoleOrOpenRole(EXECUTOR_ROLE) {
        bytes32 id = hashOperation(target, value, payload, predecessor, salt);

        _beforeCall(id, predecessor);
        _execute(target, value, payload);
        emit CallExecuted(id, 0, target, value, payload);
        _afterCall(id);
    }
```

Once ready, an executor-role holder executes the operation. The controller recorded as holding both roles does not need another independent approver. This does not bypass role checks, readiness or predecessor ordering (`src/lib/openzeppelin-contracts/contracts/governance/TimelockController.sol:356-368,418-424`).

#### Step 3 — Reach privileged target setters

Source: `src/Leverage.sol:88-93`

```solidity
    function setExecutor(address newExecutor) external {
        if (msg.sender != timelock) revert Errors.Unauthorized(msg.sender);
        if (newExecutor == address(0)) revert Errors.ZeroAddress();
        emit Events.ExecutorSet(executor, newExecutor);
        executor = newExecutor;
    }
```

The resulting target call arrives from the timelock address and satisfies the protocol's administrative gates. It can rotate the executor/quote/mark signers, change mark freshness, redirect the treasury, set the surplus split to 10,000 basis points, or clear a tripped breaker (`src/Leverage.sol:88-92`; `src/PremiumOracle.sol:110-114`; `src/MarkVerifier.sol:66-77`; `src/Liquidator.sol:210-222`; `src/Configurator.sol:244-247`).

#### Step 4 — Extend mark validity without a policy maximum

Source: `src/MarkVerifier.sol:74-78`

```solidity
    function setMarkFreshness(uint64 newWindow) external {
        if (msg.sender != timelock) revert Errors.Unauthorized(msg.sender);
        markFreshness = newWindow;
        emit Events.MarkFreshnessSet(newWindow);
    }
```

Mark freshness has no policy maximum below the `uint64` type bound. A maximum window makes a correctly signed mark effectively non-expiring over any practical protocol lifetime; it is not literally infinite (`src/MarkVerifier.sol:55-62,74-77`).

#### Step 5 — Redirect only the post-restoration surplus

Source: `src/Liquidator.sol:153-160`

```solidity
        uint256 outstanding = _buffer.outstandingAdvance(loanId);
        uint256 bufferRepaid = amount < outstanding ? amount : outstanding;
        uint256 afterAdvance = amount - bufferRepaid;
        uint256 restoredSoFar = poolRestored[loanId];
        uint256 unrestored = ledger.poolLoss(loanId) - restoredSoFar;
        uint256 restored = afterAdvance < unrestored ? afterAdvance : unrestored;
        uint256 treasuryCut = MathLib.bpsDown(afterAdvance - restored, surplusSplitBps);
        uint256 poolShare = afterAdvance - treasuryCut;
```

A 100% surplus split routes only surplus remaining after Buffer repayment and restoration of that loan's pool loss to treasury. It does not redirect the entire remittance (`src/Liquidator.sol:149-160`). A changed treasury controls that destination.

#### Step 6 — Leave users dependent on cash-gated exits

Source: `src/LatUSD.sol:293-298`

```solidity
                // What idle can cover at this moment's NAV; both roundings favor the pool, and the
                // down-down round trip guarantees fillAssets never exceeds idle.
                uint256 coverable = _convertToShares(ledger.idle(), Math.Rounding.Floor);
                fillShares = remaining <= coverable ? remaining : coverable;
                fillAssets = _convertToAssets(fillShares, Math.Rounding.Floor);
                if (fillAssets == 0) break; // idle exhausted: the line holds
```

Users observing the queue have at most the remaining configured delay before execution becomes possible. Lender shares must enter the redemption queue and wait for cash assignment; there is no general guarantee that this completes within the governance delay (`src/LatUSD.sol:225-247`; related fork exit proof `poc/src/FinalProofs.t.sol:140-150`).

### Impact and proof limits

The confirmed impact is a short response interval for privileged policy changes. No V2-specific PoC demonstrates a malicious proposal being scheduled, waiting 300 seconds and then executed against users. There is no evidence here of timelock bypass, stolen keys, an unauthorized administrator or a mainnet incident. Production severity depends on the intended deployment policy, independent approvals and users' practical escape paths; testnets may deliberately use a shorter delay.

The source register's “five minutes minus queue time” is best understood as a response-window concern. Queue completion has no demonstrated five-minute upper bound, so subtracting an assumed queue duration would be misleading. Similarly, cancellation authority held by the same controller is not an independent safety veto.

## Proof of concept

No dedicated exploit test is available. Historical configuration records show a 300-second timelock with concentrated proposer/executor/canceller roles. Local tests that impersonate the timelock demonstrate authorized configuration changes, not real scheduling or an unprivileged governance bypass.

## Relevant test excerpts

Controlled signer installation and lender funding in the shared final-proof setup:

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

The related recovery action directly impersonates the timelock, funds the executor, and asserts the resulting close. This proves target-level administrative authority only:

Source: `poc/src/FinalProofs.t.sol:274-281`

```solidity
        vm.prank(TL);
        lev.setExecutor(EXECUTOR);
        pusd.mint(EXECUTOR, principal);
        vm.startPrank(EXECUTOR);
        pusd.approve(LEVERAGE, principal);
        lev.closeLeverage(posId, principal);
        vm.stopPrank();
        assertTrue(ledger.position(posId).status == PositionStatus.Closed, "timelock restore unfreezes");
```

The related lender-exit assertion shows why requested redemption is not immediate cash exit:

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

Recorded configuration row, copied without deployment identifiers:

Source: `review-2026-10-26/outputs/config-census.md:52-52`

```text
| timelock | getMinDelay=300 |
```

Related recorded receipt, not a V2 test result:

Source: `poc/README.md:52-52`

```text
L1_RESULT={exitsBlocked:4, marginInStillOpen:true, strandedPastEpochEnd:true, restoredByTimelock:true, freezeBound:minDelay(300s)}
```

The receipt's `freezeBound:minDelay(300s)` is an interpretation in the recorded string. The test directly restores through `vm.prank(TL)` and does not establish a maximum outage or a measured 300-second governance recovery.

### Remediation direction

Choose and document production delays around detection, review and feasible lender/borrower exits. Separate proposal and execution approvals where practical, and retain an independent narrowly scoped cancellation or emergency authority. Consider longer delays for signer/custody/treasury changes and explicit safety bounds on freshness and distribution policy. Emergency recovery should restore safe operation without silently allowing immediate weakening of economic checks.

No implemented remediation or post-fix validation is established by the available evidence.
