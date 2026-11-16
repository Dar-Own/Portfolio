# Source-code-based

Project: Anonymous cross chain bridges project (BRIDGE-01)  
Category: Cross chain bridges  
Severity: Unrated — no assigned severity established in the reviewed report  
Engagement: Independent hunting + responsible disclosure (specific program channel not established)  
Chain: EVM cross-chain bridge; synthetic local home/foreign fixture, no deployment proven  
Tech stack: Solidity 0.7.5, ERC-20 custody/mint/burn, AMB messaging, JavaScript/Truffle, AMBMock  
Evidence: Historical local or fork test excerpts; scope and fixture assumptions are described below  
Remediation: Not independently verified  
Contribution: Research and disclosure case study; exclusive discovery credit is not asserted  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Failed-message refunds bypass ordinary execution limits and accounting

## Description

A sender’s failed cross-chain transfer can be refunded through the authenticated failure-recovery route even when ordinary inbound execution is disabled by a zero daily limit. The refund does not increment totalExecutedPerDay. The report’s proposed invariant treats all token releases as subject to the same execution quota; program acceptance of that invariant is not established.

This is recovery of a previously recorded transfer to its stored recipient. It does not bypass AMB authentication or authorize an arbitrary user to choose a new recipient/value. The test explicitly controls the owner settings and AMBMock delivery. It demonstrates a local accounting/limit difference, not theft of unrelated bridge funds, compromise of message consensus or a deployed exploit. Refund exemptions may be an intended policy and require protocol adjudication.

### Mechanism step by step

#### Step 1 — Apply limits to ordinary inbound delivery

Normal inbound release requires withinExecutionLimit and adds the amount to the daily execution total.

Source: `contracts/upgradeable_contracts/Home[project].sol:126-139`

```solidity
        require(withinExecutionLimit(_token, _value));
        addTotalExecutedPerDay(_token, getCurrentDay(), _value);

        uint256 valueToBridge = _value;
        uint256 fee = _distributeFee(FOREIGN_TO_HOME_FEE, _isNative, address(0), _token, valueToBridge);
        bytes32 _messageId = messageId();
        if (fee > 0) {
            emit FeeDistributed(fee, _token, _messageId);
            valueToBridge = valueToBridge.sub(fee);
        }

        _releaseTokens(_isNative, _token, _recipient, valueToBridge, _value);

        emit TokensBridged(_token, _recipient, valueToBridge, _messageId);
```

#### Step 2 — Request recovery only for a matching failed message

The permissionless request checks failed status, receiver and source mediator before sending the recovery message.

Source: `contracts/upgradeable_contracts/components/common/FailedMessagesProcessor.sol:18-26`

```solidity
    function requestFailedMessageFix(bytes32 _messageId) external {
        IAMB bridge = bridgeContract();
        require(!bridge.messageCallStatus(_messageId));
        require(bridge.failedMessageReceiver(_messageId) == address(this));
        require(bridge.failedMessageSender(_messageId) == mediatorContractOnOtherSide());

        bytes4 methodSelector = this.fixFailedMessage.selector;
        bytes memory data = abi.encodeWithSelector(methodSelector, _messageId);
        _passMessage(data, true);
```

#### Step 3 — Authenticate recovery and load the historical transfer

onlyMediator still gates the execution; the bridge loads stored token, recipient and value and rejects repeat recovery.

Source: `contracts/upgradeable_contracts/components/common/FailedMessagesProcessor.sol:34-42`

```solidity
    function fixFailedMessage(bytes32 _messageId) public onlyMediator {
        require(!messageFixed(_messageId));

        address token = messageToken(_messageId);
        address recipient = messageRecipient(_messageId);
        uint256 value = messageValue(_messageId);
        setMessageFixed(_messageId);
        executeActionOnFixedTokens(token, recipient, value);
        emit FailedMessageFixed(_messageId, token, recipient, value);
```

#### Step 4 — Release the refund without the ordinary quota hooks

The recovery implementation calls _releaseTokens directly without invoking either the normal limit check or execution-accounting update.

Source: `contracts/upgradeable_contracts/Basic[project].sol:205-211`

```solidity
    function executeActionOnFixedTokens(
        address _token,
        address _recipient,
        uint256 _value
    ) internal override {
        _releaseTokens(nativeTokenAddress(_token) == address(0), _token, _recipient, _value, _value);
    }
```

## Proof of concept

The JavaScript fixture deploys two synthetic bridges, two AMBMock instances and a minted ERC-20. Its owner sets execution limits to zero. Normal delivery of 50 tokens fails. Delivering the authenticated failure-fix message through AMBMock then refunds 50 tokens while the daily execution counter remains zero. The test source demonstrates the recovery-accounting distinction under the fixture’s message authentication; it does not demonstrate bypassing production AMB authentication.

**Proof limits:** Mocked message transport and synthetic tokens; no production relay exploit or arbitrary-recipient theft proved.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `MINIMAL_FIX_FAILED_MESSAGE_BYPASS_REPORT.md:172-205`

```text
    const failedMessage = await relayFromHome(homeToken, user2, ether('50'))
    expect(
      await executeMessage(foreignAmb, foreignBridge.address, homeBridge.address, failedMessage),
    ).to.equal(false, 'normal inbound execution must fail when executionDailyLimit = 0')

    const balanceBefore = await homeToken.balanceOf(user2)

    const fixTx = await foreignBridge.requestFailedMessageFix(failedMessage.messageId, { from: user2 })
    const fixMessage = await getMockedEvent(foreignAmb, fixTx.receipt.blockNumber)

    expect(
      await executeMessage(homeAmb, homeBridge.address, foreignBridge.address, fixMessage),
    ).to.equal(true, 'fixFailedMessage should still succeed')

    const balanceAfter = await homeToken.balanceOf(user2)
    const counterAfter = await homeBridge.totalExecutedPerDay(homeToken.address, currentDay)

    expect(await homeBridge.executionDailyLimit(homeToken.address)).to.be.bignumber.equal(
      '0',
      'executionDailyLimit(token) is still 0',
    )

    expect(balanceAfter.sub(balanceBefore)).to.be.bignumber.equal(
      ether('50'),
      'recipient received 50 tokens via fixFailedMessage',
    )

    expect(counterAfter).to.be.bignumber.equal(
      counterBefore,
      'totalExecutedPerDay did not change even though 50 tokens were released',
    )

    expect(counterAfter).to.be.bignumber.equal(
      '0',
```

Source: `contracts/upgradeable_contracts/BasicAMBMediator.sol:26-31`

```solidity
    function _onlyMediator() internal view {
        IAMB bridge = bridgeContract();
        require(msg.sender == address(bridge));
        require(bridge.messageSender() == mediatorContractOnOtherSide());
    }

```
