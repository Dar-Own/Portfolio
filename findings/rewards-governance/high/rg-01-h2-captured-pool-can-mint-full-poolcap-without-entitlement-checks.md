# Source-code-based

Project: Anonymous reward issuance and governance protocol (RG-01)
Category: Reward issuance and governance
Severity: High (final report assignment)
Engagement: Private engagement
Chain: EVM; Sepolia release attestation is separate from the historical local PoCs
Tech stack: Solidity, OpenZeppelin, ERC-20, Merkle proofs, Governor/Timelock, Trustline SDK, Hardhat, TypeScript
Evidence: Original local test source; mechanism assertions shown below
Remediation: Remediated — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Captured pool can mint full poolCap without entitlement checks

## Description

Governance-admitted malicious/compromised pool holds mint authority, sufficient unclaimed reservations and remaining global capacity → pool calls mint for chosen recipients → minter verifies caller/pool/status/caps but not reward entitlement → fraudulent issuance consumes the admitted pool’s budget.

Fraudulent issuance within admitted pool/global caps; configured budgets 100M NMRP1 / 200M contributor.

Remediated at commit a79e61b, with the hash-helper entrypoint corrected at 61fd881. The deployment script compares each pool's live runtime bytecode against externally supplied expected hashes at scripts/deploy.ts:737-760 before review linkage and whitelisting at lines 764-809. scripts/computeReviewedPoolHashes.cjs:60-97 accounts for immutable-linked contributor-pool runtime bytecode, and package.json:2,8 invokes it through a CommonJS-compatible .cjs entrypoint. The hash control is strong only when expected hashes are independently reproduced and authenticated from the audited release. [project]MintContract.mint still trusts admitted pools to enforce entitlement correctness; compromise of an admitted pool can therefore cause fraudulent issuance within its pool and global caps.

### Mechanism step by step

#### Step 1 — mint has role + self-scope + caps, NO entitlement/compliance check

nothing ties `to`/`amount` to any proof; the only ceiling is the pool's own poolCap.

Source: `contracts/[project]MintContract.sol:171-198` (commit 95fc21c).

```solidity
    function mint(
        address poolAddress,
        address to,
        uint256 amount
    ) public nonReentrant onlyRole(AUTHORIZED_PARTY_ROLE) {
        require(to != spendReceiver, "Cannot mint to spend receiver");
        require(
            msg.sender == poolAddress,
            "Pool mint: only the pool can mint for itself"
        );
        RewardContract storage pool = rewardContracts[poolAddress];
        require(pool.contractAddress != address(0), "Pool not found");
        require(pool.active, "Pool not active");
        require(pool.unclaimedRewards >= amount, "Insufficient unclaimed");

        pool.unclaimedRewards -= amount;
        pool.claimedRewards += amount;
        totalUnclaimedRewards -= amount;

        // ✅ Enforce combined cap
        require(
            pool.claimedRewards + pool.unclaimedRewards <= pool.poolCap,
            "Mint cap exceeded"
        );

        // _mint from ERC20Capped will revert if totalSupply()+amount > START_HARD_CAP
        _mint(to, amount);
        emit TokensMinted(to, amount);
```

#### Step 2 — whitelisting grants the mint role

any address admin whitelists receives AUTHORIZED_PARTY_ROLE and becomes a minting pool.

Source: `contracts/[project]MintContract.sol:252-252` (commit 95fc21c).

```solidity
        _grantRole(AUTHORIZED_PARTY_ROLE, contractAddress);
```

## Proof of concept

The administrator whitelists an EOA as a pool stand-in. That admitted pool reserves and mints its full cap to itself. This proves that entitlement validation is delegated to the admitted pool; it does not prove an unauthenticated path to mint authority.

The tests deploy fresh contracts with synthetic actors. Accounting and governance fixtures use an always-allowing validation mock; M9/M10 use a configurable mock. An EOA admitted as a pool stands in for a governance-approved pool contract where the boundary is tested directly. These are local tests, despite the original configuration requiring a Sepolia fork.

Test references:

- `PoCs/helpers/deploy.ts`
- `PoCs/pool-capture-auth/H2_malicious_pool_mints_full_poolCap.test.ts`
- `PoCs/pool-capture-auth/H2_malicious_pool_mints_full_poolCap.test.ts:18-27`

## Relevant test excerpts

Source: `PoCs/pool-capture-auth/H2_malicious_pool_mints_full_poolCap.test.ts:18-27`.

```typescript
  it("mints full poolCap directly through minter", async () => {
    await core.[project].connect(core.attacker).updateUnclaimedRewards(pool, poolCap);

    // Minter trusts any whitelisted pool; no leaf or recipient entitlement is checked here.
    await core.[project].connect(core.attacker).mint(pool, attacker, poolCap);

    expect(await core.[project].balanceOf(attacker)).to.equal(poolCap);
    expect((await core.[project].rewardContracts(pool)).claimedRewards).to.equal(poolCap);
  });
});
```
