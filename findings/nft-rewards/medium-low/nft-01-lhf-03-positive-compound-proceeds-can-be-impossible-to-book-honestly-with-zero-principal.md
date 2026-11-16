# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Medium to Low ceiling; accounting/liveness candidate with privileged recovery  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Historical local or fork test excerpts; scope and fixture assumptions are described below  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Positive compound proceeds can be impossible to book honestly with zero principal

## Description

A conversion-bearing round can have zero principal weights. A positive conversion report then cannot attribute its capital remainder. Confirming zero clears the liveness gate but omits the modeled positive conversion from historical compound accounting; an administrator can restore current principal through manual adjustment without restoring the original compound history.

Prerequisites and limits: The fixture constructs the zero-weight round and models 100 xPHAR converted off chain; it performs no real exchange conversion. The operator can clear the duty with confirmZero, and DEFAULT_ADMIN can restore balances. This is not a demonstrated direct theft or permanent inability to process all later rounds.

### Mechanism step by step

#### Step 1 — A round produces conversion work with zero weights

Conversion split allocates diverted amounts first and then attributes the remainder by capital weight. A nonzero unattributed remainder sets noWeights.

Source: `contracts/contracts/libraries/[project]Math.sol:150-181`.

```solidity
    function splitConversion(
        uint128 xphar,
        uint256 toConvertAvax,
        uint256[] memory divertedAvax,
        uint256[] memory weights
    )
        internal
        pure
        returns (
            uint256[] memory divertedParts,
            uint256[] memory capitalShares,
            uint256 divertedTotal,
            uint256 rest,
            bool noWeights
        )
    {
        uint256 count = divertedAvax.length;
        divertedParts = new uint256[](count);
        for (uint256 i; i < count; ++i) {
            uint256 own = divertedAvax[i];
            if (own == 0) continue;
            uint256 part = Math.mulDiv(xphar, own, toConvertAvax);
            if (part == 0) continue;
            divertedParts[i] = part;
            divertedTotal += part;
        }

        rest = uint256(xphar) - divertedTotal;

        uint256 unattributed;
        (capitalShares, unattributed) = attributeProRata(rest, weights);
        noWeights = unattributed != 0;
```

#### Step 2 — An honest positive report is refused

The writer rejects the split before crediting compound or diverted accounting.

Source: `contracts/contracts/core/[project]CompoundReporting.sol:69-90`.

```solidity
    function _attribute(uint32 vaultId, uint32 roundId, uint128 xphar, uint256 toConvert) private {
        CoreStorage storage $ = _getCoreStorage();

        Snapshot memory s = _loadSnapshot($, vaultId, roundId);

        (
            uint256[] memory divertedParts,
            uint256[] memory capitalShares,
            uint256 divertedTotal,
            uint256 rest,
            bool noWeights
        ) = [project]Math.splitConversion(xphar, toConvert, s.divertedAvax, s.weights);
        if (noWeights) revert NoWeights();
        _creditDivertedParts($, vaultId, roundId, s, divertedParts);
        _creditCapitalShares($, vaultId, roundId, s, capitalShares);

        VaultRound storage vr = $.vaultRounds[vaultId][roundId];
        vr.divertedXphar += SafeCast.toUint128(divertedTotal);
        vr.compoundXphar += SafeCast.toUint128(rest);
        VaultStats storage vs = $.vaultStats[vaultId];
        vs.totalDivertedXphar += SafeCast.toUint128(divertedTotal);
        vs.totalCompoundXphar += SafeCast.toUint128(rest);
```

#### Step 3 — Confirming zero bypasses attribution and clears the duty

A zero amount with explicit confirmation passes the check and skips _attribute; marking reported decrements the pending count once.

Source: `contracts/contracts/core/[project]CompoundReporting.sol:19-62`.

```solidity
    function reportCompound(uint32 vaultId, uint32 roundId, uint128 xphar, bool confirmZero, string calldata memo)
        external
        onlyRoleOrAdmin(OPERATOR_ROLE)
        nonReentrant
    {
        uint256 toConvert = _checkReport(vaultId, roundId, xphar, confirmZero);

        if (xphar != 0) _attribute(vaultId, roundId, xphar, toConvert);

        _markReported(vaultId, roundId, toConvert);

        uint128 cumulative = _roundXphar(vaultId, roundId);
        emit CompoundReported(vaultId, roundId, xphar, cumulative, memo);
    }

    function _checkReport(uint32 vaultId, uint32 roundId, uint128 xphar, bool confirmZero)
        internal
        view
        returns (uint256 toConvert)
    {
        _requireFlagClear(PAUSE_REPORTING);

        Vault storage v = _vault(vaultId);

        _requireNotShort(v);

        if (roundId == 0 || roundId > v.roundCount) revert UnknownRound(vaultId, roundId);

        VaultRound storage vr = _getCoreStorage().vaultRounds[vaultId][roundId];
        toConvert = uint256(vr.compoundAvax) + vr.collectionDivertedAvax + vr.vaultDivertedAvax;

        if (toConvert == 0) {
            if (xphar != 0 || !confirmZero) revert NoCompoundExpected();
        } else if (xphar == 0 && !confirmZero) {
            revert ZeroAmount();
        }
    }

    function _markReported(uint32 vaultId, uint32 roundId, uint256 toConvert) private {
        CoreStorage storage $ = _getCoreStorage();
        if ($.compoundReported[vaultId][roundId]) return;
        $.compoundReported[vaultId][roundId] = true;
        if (toConvert != 0) $.vaults[vaultId].pendingCompoundRounds -= 1;
    }
```

## Proof of concept

The local test models 100 xPHAR of off-chain conversion in a zero-weight round. An honest positive report reverts, while confirmZero clears the pending duty. Administrative adjustXphar restores current balances/manual-in counters, but historical round and cumulative compound attribution remain zero. The result is incorrect historical attribution after privileged liveness recovery, not a real exchange conversion or direct theft.

**Proof limits:** Constructed conversion value; privileged recovery remains available but does not repair historical attribution.

The excerpts below preserve historical assertions and recorded outcomes; they are not a fresh execution result.

## Relevant test excerpts

Source: `contracts/audit-tests/test/unit/compound-noweights-settlement.test.ts:34-50`.

```typescript
async function noWeightsWorld(): Promise<NoWeightsWorld> {
  const f = await deployCoreFixture();
  const walletAddress = await f.wallet.getAddress();
  await mined(f.admin.sendTransaction({ to: walletAddress, value: WALLET_FUNDING }) as Promise<ContractTransactionResponse>);
  await mined(f.core.createVault(NO_COLLECTION_SPLIT, walletAddress, 0n, ROUND_INTERVAL, ELIGIBILITY_DELAY, f.anchor));

  const base = BigInt((await f.lens.getVault(VAULT_1)).lastCutoff);
  const col = await addCollectionTo(f, VAULT_1, "No weights settlement", "NWSET");
  await activateTokens(f, col, f.holder, [1n, 2n, 3n, 4n, 5n]);

  const { oracle } = await processOracleRound(f, VAULT_1, ROUND_ID, base + ROUND_INTERVAL, E18, NO_COLLECTION_SPLIT, [
    { eligible: 0n, xphar: 0n },
  ]);
  expect(oracle.compoundAvaxToConvert, "the fixture leaves a positive conversion base").to.equal(8n * 10n ** 17n);
  expect((await f.lens.getCollectionRound(col.address, ROUND_ID)).xpharAtRound, "every snapshot weight is zero").to.equal(0n);

  return { ...f, collection: col.address, base, toConvertAvax: oracle.compoundAvaxToConvert };
```

Source: `contracts/audit-tests/test/unit/compound-noweights-settlement.test.ts:62-96`.

```typescript
  it("measures the zero-confirmation settlement and the admin-only manual reconstruction", async function () {
    const f = await networkHelpers.loadFixture(noWeightsWorld);

    // 1. A real positive conversion result hits the writer's NoWeights guard.
    const positiveReport = as(f.core, f.operator).reportCompound(VAULT_1, ROUND_ID, REAL_CONVERSION_XPHAR, false, "actual positive conversion");
    await expect(positiveReport).to.be.revertedWithCustomError(f.core, "NoWeights");

    let positiveReportSelector = "";
    try {
      await as(f.core, f.operator).reportCompound.staticCall(VAULT_1, ROUND_ID, REAL_CONVERSION_XPHAR, false, "actual positive conversion");
      throw new Error("positive report unexpectedly succeeded");
    } catch (error) {
      positiveReportSelector = revertData(error).slice(0, 10);
    }
    expect(positiveReportSelector, "the positive report reverts with the NoWeights selector").to.equal(NO_WEIGHTS_SELECTOR);
    expect((await f.lens.getVault(VAULT_1)).pendingCompoundRounds, "the failed report leaves the round pending").to.equal(1n);

    // 2. Confirming zero bypasses attribution and settles the pending-round gate.
    const zeroMemo = "confirmed zero despite the off-chain positive conversion";
    const zeroReceipt = await mined(as(f.core, f.operator).reportCompound(VAULT_1, ROUND_ID, 0n, true, zeroMemo));
    const zeroReports = eventsOf(f.core, zeroReceipt, "CompoundReported");
    expect(zeroReports.length, "one zero CompoundReported event").to.equal(1);
    expect(zeroReports[0]!.args.xphar, "the only recorded compound amount is zero").to.equal(0n);
    expect((await f.lens.getVault(VAULT_1)).pendingCompoundRounds, "zero confirmation clears the pending conversion").to.equal(0n);
    expect(await f.lens.isCompoundReported(VAULT_1, ROUND_ID), "the conversion round is permanently marked reported").to.equal(true);

    const zeroVaultRound = await f.lens.getVaultRound(VAULT_1, ROUND_ID);
    const zeroCollectionRound = await f.lens.getCollectionRound(f.collection, ROUND_ID);
    const zeroVaultStats = await f.lens.getVaultStats(VAULT_1);
    const zeroCollectionStats = await f.lens.getCollectionStats(f.collection);
    expect(zeroVaultRound.compoundXphar, "settlement does not record the actual conversion in the vault round").to.equal(0n);
    expect(zeroCollectionRound.compoundXphar, "nor in the collection round").to.equal(0n);
    expect(zeroVaultStats.totalCompoundXphar, "nor in vault compound totals").to.equal(0n);
    expect(zeroCollectionStats.compoundXphar, "nor in collection compound totals").to.equal(0n);

```

Source: `contracts/audit-tests/test/unit/compound-noweights-settlement.test.ts:125-145`.

```typescript
    const repairedVaultRound = await f.lens.getVaultRound(VAULT_1, ROUND_ID);
    const repairedCollectionRound = await f.lens.getCollectionRound(f.collection, ROUND_ID);
    expect(repairedVault.xpharTotal, "manual entry restores the current vault balance").to.equal(REAL_CONVERSION_XPHAR);
    expect(repairedCollection.xphar, "manual entry restores the current collection balance").to.equal(REAL_CONVERSION_XPHAR);
    expect(repairedVaultStats.manualInXphar, "vault records a manual-in counter").to.equal(REAL_CONVERSION_XPHAR);
    expect(repairedCollectionStats.manualInXphar, "collection records a manual-in counter").to.equal(REAL_CONVERSION_XPHAR);

    const repairedFieldsMatch =
      repairedVaultRound.compoundXphar === REAL_CONVERSION_XPHAR &&
      repairedCollectionRound.compoundXphar === REAL_CONVERSION_XPHAR &&
      repairedVaultStats.totalCompoundXphar === REAL_CONVERSION_XPHAR &&
      repairedCollectionStats.compoundXphar === REAL_CONVERSION_XPHAR;
    expect(repairedFieldsMatch, "manual reconstruction cannot repair historical compound-attribution counters").to.equal(false);

    console.log(
      `adjustXphar emitted XpharAdjusted(id=${adjusted[0]!.args.adjustmentId}, delta=${adjusted[0]!.args.delta}, newXphar=${adjusted[0]!.args.newXphar}) ` +
        `and XpharChanged(source=${changed[0]!.args.source}, delta=${changed[0]!.args.delta}, refId=${changed[0]!.args.refId}).`,
    );
    console.log(
      `manual reconstruction restores collection/vault xPHAR and manualInXphar, but leaves round compoundXphar and compound totals at ${repairedVaultRound.compoundXphar}.`,
    );
```
