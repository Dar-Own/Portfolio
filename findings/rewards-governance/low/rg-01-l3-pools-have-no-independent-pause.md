# Source-code-based

Project: Anonymous reward issuance and governance protocol (RG-01)
Category: Reward issuance and governance
Severity: Low (final report assignment)
Engagement: Private engagement
Chain: EVM; Sepolia release attestation is separate from the historical local PoCs
Tech stack: Solidity, OpenZeppelin, ERC-20, Merkle proofs, Governor/Timelock, Trustline SDK, Hardhat, TypeScript
Evidence: Source/configuration analysis
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Pools have no independent pause

## Description

Incident affects one reward pool → administrator needs isolate its claim/submit methods → original pool has no independent pause → containment requires broader token stop.

Cannot independently contain one affected pool while other reward paths operate.

Resolved at commit 1c2c2fd in both reward pools. NMRP1 and [project]ContributorAllocations inherit Pausable at contracts/NMRP1.sol:39 and contracts/[project]ContributorAllocations.sol:28. Submission and claim entry points use whenNotPaused, and each pool exposes separate admin-only pause and unpause functions at contracts/NMRP1.sol:302-307 and contracts/[project]ContributorAllocations.sol:149-154. Governance can contain one pool without stopping the other.

### Mechanism step by step

#### Step 1 — Pool claim depends on the shared token path

Claim execution reaches the common minter. The original pool has no independent pause control to isolate this claim surface.

Source: `contracts/mintUscyRewards.sol:228-237` (commit 95fc21c).

```solidity
            if (!hasClaimsStarted[uuidHash]) {
                hasClaimsStarted[uuidHash] = true;
                emit ClaimsStarted(uuidHash);
            }

            // Mark as claimed
            hasClaimed[uuidHash][user][eid] = true;

            // Mint to the user
            mintingContract.mint(poolAddresses[i], user, amounts[i]);
```

#### Step 2 — The available pause acts on the common minter

The emergency administrator can pause the shared minter, affecting other pools too. The limitation is incident-containment granularity, not unauthorized minting.

Source: `contracts/[project]MintContract.sol:206-212` (commit 95fc21c).

```solidity
    function pause() external onlyRole(DEFAULT_ADMIN_ROLE) {
        _pause();
    }

    function unpause() external onlyRole(DEFAULT_ADMIN_ROLE) {
        _unpause();
    }
```

## Proof of concept

The source shows the shared minter pause and the absence of an independent pool pause in the affected revision. No dedicated exploit test is available; the finding concerns emergency-containment granularity.

## Relevant test excerpts

No finding-specific executable test excerpt is available. The source excerpts above are the supporting evidence.
