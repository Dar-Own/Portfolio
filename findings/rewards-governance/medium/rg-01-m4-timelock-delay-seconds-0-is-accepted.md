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

# TIMELOCK_DELAY_SECONDS=0 is accepted

## Description

Deployer configures TIMELOCK_DELAY_SECONDS=0 → validation rejects only invalid number → zero-delay timelock accepted → governance review delay disappears.

Administrative proposals can execute without intended response window under that deployment config.

Resolved at commit 1c2c2fd and strengthened at 968adfd. The deployment script rejects delays below 48 hours at scripts/deploy.ts:20-33. Independently, [project]GoverningCouncil.initialize rejects a timelock whose getMinDelay() is below 48 hours at contracts/[project]GoverningCouncil.sol:66-70. A zero or short-delay governor deployment therefore fails even outside the deployment script.

### Mechanism step by step

#### Step 1 — only NaN is rejected

Number("0")===0 passes Number.isNaN, so 0 flows through as the timelock delay.

Source: `scripts/deploy.ts:8-12` (commit 95fc21c).

```typescript
const TIMELOCK_DELAY = Number(process.env.TIMELOCK_DELAY_SECONDS);

if (Number.isNaN(TIMELOCK_DELAY)) {
  throw new Error("TIMELOCK_DELAY_SECONDS must be set (in seconds), e.g. 172800");
}
```

The local helper hardcodes delay zero. That proves the configuration is possible, not that a live deployment used it.

#### Step 2 — Pass the accepted zero delay into the deployed timelock

The validated value is supplied directly to the initializer. No later minimum-delay check in this deployment sequence restores a response window.

Source: `scripts/deploy.ts:104-111` (commit 95fc21c).

```typescript
  console.log("🛠 Deploying TimelockController...");
  const TimelockFactory = await ethers.getContractFactory("TimelockControllerUpgradeable");
  const timelock = await upgrades.deployProxy(
    TimelockFactory,
    [TIMELOCK_DELAY, [deployerAddress], [deployerAddress], deployerAddress],
    { initializer: "initialize" }
  );
  await timelock.waitForDeployment();
```

## Proof of concept

The local helper deploys a timelock with delay zero and asserts getMinDelay()==0. Combined with the script’s acceptance of zero, this establishes a permitted configuration. It does not establish the delay of a live deployment.

The tests deploy fresh contracts with synthetic actors. Accounting and governance fixtures use an always-allowing validation mock; M9/M10 use a configurable mock. An EOA admitted as a pool stands in for a governance-approved pool contract where the boundary is tested directly. These are local tests, despite the original configuration requiring a Sepolia fork.

Test references:

- `PoCs/deploy-governance/M4_zero_timelock_delay.test.ts`
- `PoCs/deploy-governance/M4_zero_timelock_delay.test.ts:6-10`
- `PoCs/helpers/deploy.ts`

## Relevant test excerpts

Source: `PoCs/deploy-governance/M4_zero_timelock_delay.test.ts:6-10`.

```typescript
  it("deploys with minDelay equal to zero", async () => {
    const deployed = await deployGovernance(1);
    expect(await deployed.timelock.getMinDelay()).to.equal(0n);
  });
});
```
