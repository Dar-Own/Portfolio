# Source-code-based

Project: Anonymous reward issuance and governance protocol (RG-01)
Category: Reward issuance and governance
Severity: Low (final report assignment)
Engagement: Private engagement
Chain: EVM; Sepolia release attestation is separate from the historical local PoCs
Tech stack: Solidity, OpenZeppelin, ERC-20, Merkle proofs, Governor/Timelock, Trustline SDK, Hardhat, TypeScript
Evidence: Original local test source; partial proof with limits below
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Timelock lacks AUTHORIZED_PARTY_ROLE on minter

## Description

Deployment grants timelock DEFAULT_ADMIN but not operational minter role → governance invokes updateAndAdjust → role check rejects → normal cap operation unavailable until role correction.

Governance role wiring blocks cap changes; admin can grant itself missing role.

Resolved at commit 894c36f. Global-cap adjustment is restricted to CAP_MANAGER_ROLE at contracts/[project]MintContract.sol:23 and lines 164-191. The deployment script passes the actual governance timelock to the minter constructor at scripts/deploy.ts:426-435, aborts unless that timelock holds the role at lines 922-925, and removes the deployer's copy at lines 731-735. The governance suite confirms that an unauthorized caller cannot invoke updateAndAdjust and that a timelock-executed proposal can update the moving average and supply cap.

### Mechanism step by step

#### Step 1 — Constructor grants an operational role to its first argument

The constructor distinguishes its first argument from the separate deployer argument, but grants operational authority only to the first.

Source: `contracts/[project]MintContract.sol:79-98` (commit 95fc21c).

```solidity
    constructor(
        address timelockController,
        address deployer,
        address spendReceiver_
    )
        ERC20("[project name redacted]", "[project]")
        ERC20Capped(START_HARD_CAP)
        ERC20Permit("[project name redacted]")
    {
        require(timelockController != address(0), "Invalid timelock address");
        require(deployer != address(0), "Invalid deployer address");
        require(spendReceiver_ != address(0), "Invalid spend receiver");

        _grantRole(DEFAULT_ADMIN_ROLE, timelockController);
        _grantRole(DEFAULT_ADMIN_ROLE, deployer);
        _grantRole(AUTHORIZED_PARTY_ROLE, timelockController);

        supplyCap = START_HARD_CAP;
        spendReceiver = spendReceiver_;
    }
```

#### Step 2 — Deployment supplies the deployer in that position

The supplied first argument is the deployer. A later DEFAULT_ADMIN grant to the timelock does not implicitly confer AUTHORIZED_PARTY_ROLE.

Source: `scripts/deploy.ts:125-132` (commit 95fc21c).

```typescript
  const timelockController = deployer.address;

  const [project]Factory = await ethers.getContractFactory("[project]MintContract");
  const [project]MintContract = await [project]Factory.deploy(
    timelockController,
    deployer.address,
    [project]SpendReceiverAddress
  );
```

The final report labels its first L1 excerpt as audited commit 95fc21c, but that excerpt includes later CAP_MANAGER_ROLE grants absent from the actual Git object. The excerpts above use the verified historical constructor and deployment arguments instead. The timelock can recover through its admin power.

## Proof of concept

The helper grants DEFAULT_ADMIN_ROLE to a timelock without granting the operational minter role. The role-state assertion establishes that these grants are independent. It does not execute the production deployment script; the timelock’s admin authority can grant the missing role.

The tests deploy fresh contracts with synthetic actors. Accounting and governance fixtures use an always-allowing validation mock; M9/M10 use a configurable mock. An EOA admitted as a pool stands in for a governance-approved pool contract where the boundary is tested directly. These are local tests, despite the original configuration requiring a Sepolia fork.

Test references:

- `PoCs/deploy-governance/L1_timelock_missing_minter_auth.test.ts`
- `PoCs/deploy-governance/L1_timelock_missing_minter_auth.test.ts:7-19`
- `PoCs/helpers/deploy.ts`

## Relevant test excerpts

Source: `PoCs/deploy-governance/L1_timelock_missing_minter_auth.test.ts:7-19`.

```typescript
  it("grants timelock admin but not AUTHORIZED_PARTY_ROLE", async () => {
    const [deployer] = await ethers.getSigners();
    const core = await deployCore({ minterTimelock: await deployer.getAddress() });
    const timelock = await core.timelock.getAddress();
    const admin = await core.[project].DEFAULT_ADMIN_ROLE();
    const auth = await core.[project].AUTHORIZED_PARTY_ROLE();

    await core.[project].grantRole(admin, timelock);

    expect(await core.[project].hasRole(admin, timelock)).to.equal(true);
    expect(await core.[project].hasRole(auth, timelock)).to.equal(false);
  });
});
```
