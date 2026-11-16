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

# Unbounded reward arrays

## Description

Caller supplies an excessively large claimRewards array → no application-level length bound limits per-entry work → transaction can exceed its gas budget and revert atomically.

Caller/integrator batch availability failure; no demonstrated cross-user theft.

Resolved at commit 894c36f. NMRP1 defines MAX_BATCH_SIZE = 100 at contracts/NMRP1.sol:43-46, and claimRewards rejects larger arrays before entering the per-entry loop at lines 236-249. Larger claims can be submitted through multiple bounded transactions. The repository does not yet contain tests proving that 100 realistic claims fit the intended production gas budget or that 101 entries revert without state changes.

### Mechanism step by step

#### Step 1 — Backend supplies an unbounded reservation batch

The reservation loop processes every supplied entry without an application-level batch limit.

Source: `contracts/mintUscyRewards.sol:147-159` (commit 95fc21c).

```solidity
        // Allow setting a root only once (or re-setting before any claim)
        require(
            merkleRoots[uuidHash] == bytes32(0),
            "Set-once policy (cannot reset even before claims)"
        );
        for (uint i = 0; i < mintData.length; i++) {
            mintInfoData[mintData[i].contractAddress] = mintData[i];
            emit MintInfoUpdated(mintData[i].contractAddress, mintData[i]);
        }

        merkleRoots[uuidHash] = newRoot;
        emit RewardSubmitted(totalReward, newRoot, uuidHash);
        mintingContract.updateUnclaimedRewards(address(this), totalReward);
```

#### Step 2 — Claim processing repeats mint work for the supplied length

Each entry incurs proof/accounting/mint work. An excessive batch exhausts its transaction gas and reverts atomically; this establishes caller or integrator availability loss, not a persistent global lock.

Source: `contracts/mintUscyRewards.sol:206-237` (commit 95fc21c).

```solidity
        );

        uint256 totalMinted = 0;

        for (uint i = 0; i < n; i++) {
            bytes32 uuidHash = uuidHashes[i];
            uint256 eid = earnedIds[i];
            bytes32 root = merkleRoots[uuidHash];
            require(root != bytes32(0), "No merkle root for this UUID");
            // Prevent double‐claim of the same entry
            require(!hasClaimed[uuidHash][user][eid], "Already claimed");
            require(amounts[i] > 0, "Zero amount");
            // Verify the Merkle proof
            bytes32 leaf = keccak256(
                abi.encode(user, accountIDHashes[i], amounts[i])
            );
            require(
                MerkleProof.verify(proofs[i], root, leaf),
                "Invalid Merkle proof"
            );

            // Mark that claims have started for this UUID (freezes root)
            if (!hasClaimsStarted[uuidHash]) {
                hasClaimsStarted[uuidHash] = true;
                emit ClaimsStarted(uuidHash);
            }

            // Mark as claimed
            hasClaimed[uuidHash][user][eid] = true;

            // Mint to the user
            mintingContract.mint(poolAddresses[i], user, amounts[i]);
```

## Proof of concept

The source shows loops over caller-supplied arrays without an application batch bound. No dedicated gas-exhaustion test is available, and permanent denial to other users is not established.

## Relevant test excerpts

No finding-specific executable test excerpt is available. The source excerpts above are the supporting evidence.
