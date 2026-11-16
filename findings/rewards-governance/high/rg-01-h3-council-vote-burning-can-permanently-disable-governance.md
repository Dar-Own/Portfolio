# Source-code-based

Project: Anonymous reward issuance and governance protocol (RG-01)
Category: Reward issuance and governance
Severity: High (final report assignment)
Engagement: Private engagement
Chain: EVM; Sepolia release attestation is separate from the historical local PoCs
Tech stack: Solidity, OpenZeppelin, ERC-20, Merkle proofs, Governor/Timelock, Trustline SDK, Hardhat, TypeScript
Evidence: Source/configuration analysis
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Council vote burning can permanently disable governance

## Description

Council holder at three-vote supply invokes inherited public burn → supply drops below the three-vote quorum floor → no normal governance proposal can restore voting supply.

Governance permanently unavailable through normal voting after supply falls below quorum.

Resolved at commit a79e61b. [project]CouncilToken no longer inherits or initializes ERC20BurnableUpgradeable at contracts/[project]CouncilToken.sol:4-16 and lines 45-58. Council holders therefore have no public burn or burnFrom entrypoint. Vote destruction remains internal to governance-controlled removeWhitelistedAddress at lines 103-117.

### Mechanism step by step

#### Step 1 — Holder destroys the voting supply

The inherited burn extension exposes token burning to holders; the historical implementation then initialized that extension.

Source: `contracts/[project]CouncilToken.sol:12-19` (commit 0c74a04).

```solidity
contract [project]CouncilToken is
    Initializable,
    ERC20Upgradeable,
    ERC20BurnableUpgradeable,
    ERC20PausableUpgradeable,
    ERC20PermitUpgradeable,
    AccessControlUpgradeable
{
```

#### Step 2 — Quorum remains above the surviving votes

The floor remains three votes even after burning reduces usable supply below three. Normal governance cannot pass its own recovery proposal.

Source: `contracts/[project]GoverningCouncil.sol:107-117` (commit 0c74a04).

```solidity
    function quorum(
        uint256 blockNumber
    )
        public
        view
        override(GovernorUpgradeable, GovernorVotesQuorumFractionUpgradeable)
        returns (uint256)
    {
        uint256 q = super.quorum(blockNumber);
        if (q < 3) return 3;
        return q;
```

## Proof of concept

Historical inheritance and quorum logic establish the vote-burning path. No finding-specific executable original exploit test is available in this case study.

## Relevant test excerpts

No finding-specific executable test excerpt is available. The source excerpts above are the supporting evidence.
