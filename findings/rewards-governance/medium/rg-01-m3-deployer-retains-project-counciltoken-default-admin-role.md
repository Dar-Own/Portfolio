# Source-code-based

Project: Anonymous reward issuance and governance protocol (RG-01)
Category: Reward issuance and governance
Severity: Medium (final report assignment)
Engagement: Private engagement
Chain: EVM; Sepolia release attestation is separate from the historical local PoCs
Tech stack: Solidity, OpenZeppelin, ERC-20, Merkle proofs, Governor/Timelock, Trustline SDK, Hardhat, TypeScript
Evidence: Original local test source; partial proof with limits below
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Deployer retains [project]CouncilToken.DEFAULT_ADMIN_ROLE

## Description

Deployment initializes council token with deployer admin → cleanup list omits token → no renunciation → deployer retains DEFAULT_ADMIN_ROLE beyond handoff.

Deployer can retain council-token administration outside governance.

Resolved at commit 1c2c2fd. [project]CouncilToken is included in accessControlledContracts at scripts/deploy.ts:507-517. The deployment handoff ensures the timelock holds DEFAULT_ADMIN_ROLE at lines 699-709 and removes the deployer's role at lines 722-733. The later verification target list still omits the council token, so a dedicated final assertion remains useful hardening, but the deployment handoff performs the required renunciation.

### Mechanism step by step

#### Step 1 — initializer grants deployer (msg.sender) admin

Deployment initializes council token with deployer admin

Source: `contracts/[project]CouncilToken.sol:57-58` (commit 95fc21c).

```solidity
        _grantRole(DEFAULT_ADMIN_ROLE, _timelockController);
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender); // Grant deployer DEFAULT_ADMIN_ROLE
```

#### Step 2 — renounce loop iterates only [minter, uscy, contributor]

council token is absent from accessControlledContracts.

Source: `scripts/deploy.ts:204-208` (commit 95fc21c).

```typescript
  const accessControlledContracts: AccessControlLike[] = [
    [project]MintAccess,
    uscyMintAccess,
    contributorAllocationsAccess,
  ];
```

#### Step 3 — deployer renounce only over that list

no renunciation

Source: `scripts/deploy.ts:333-341` (commit 95fc21c).

```typescript
  for (const contract of accessControlledContracts) {
    if (await contract.hasRole(authorizedPartyRole, deployerAddress)) {
      await (await contract.renounceRole(authorizedPartyRole, deployerAddress)).wait();
      console.log(`✅ Revoked AUTHORIZED_PARTY_ROLE on ${await contract.getAddress()}`);
    }
    if (await contract.hasRole(defaultAdminRole, deployerAddress)) {
      await (await contract.renounceRole(defaultAdminRole, deployerAddress)).wait();
      console.log(`✅ Revoked DEFAULT_ADMIN_ROLE on ${await contract.getAddress()}`);
    }
```

#### Step 4 — what that retained admin can do (reshape the electorate)

deployer retains DEFAULT_ADMIN_ROLE beyond handoff.

Source: `contracts/[project]CouncilToken.sol:96-96` (commit 95fc21c).

```solidity
        _mint(account, 1);
```

Source: `contracts/[project]CouncilToken.sol:112-112` (commit 95fc21c).

```solidity
        _burn(account, balanceOf(account));
```

The PoC verifies the initializer grant. The omitted deployment cleanup is separately source-backed; the helper does not execute the production deployment script.

## Proof of concept

The governance helper initializes the council token and asserts that the deployer retains DEFAULT_ADMIN_ROLE. It does not run the production deployment cleanup loop. The omitted cleanup is a separate source observation; live role retention is not established by the helper.

The tests deploy fresh contracts with synthetic actors. Accounting and governance fixtures use an always-allowing validation mock; M9/M10 use a configurable mock. An EOA admitted as a pool stands in for a governance-approved pool contract where the boundary is tested directly. These are local tests, despite the original configuration requiring a Sepolia fork.

Test references:

- `PoCs/deploy-governance/M3_council_token_deployer_admin_retained.test.ts`
- `PoCs/deploy-governance/M3_council_token_deployer_admin_retained.test.ts:6-13`
- `PoCs/helpers/deploy.ts`

## Relevant test excerpts

Source: `PoCs/deploy-governance/M3_council_token_deployer_admin_retained.test.ts:6-13`.

```typescript
  it("keeps deployer admin on council token", async () => {
    const deployed = await deployGovernance(1);
    const admin = await deployed.token.DEFAULT_ADMIN_ROLE();
    const deployer = await deployed.deployer.getAddress();

    expect(await deployed.token.hasRole(admin, deployer)).to.equal(true);
  });
});
```
