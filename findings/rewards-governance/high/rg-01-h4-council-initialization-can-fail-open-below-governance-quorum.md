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

# Council initialization can fail open below governance quorum

## Description

Deployer encounters failed initial whitelist/mint operations → catch/log suppresses failure → deployment proceeds and hands administration to governance without verifying three distinct council wallets each holding a usable voting token → usable voting supply is below quorum.

Governance can launch unable to pass corrective proposals.

Resolved at commit a79e61b. The deployment script requires three unique council wallets at scripts/deploy.ts:514-524, converts every initialization failure into a deployment error at lines 528-571, and then verifies that each wallet is whitelisted with exactly one vote and total voting supply is at least three at lines 574-588. These checks execute before governor deployment and privileged role handoff.

### Mechanism step by step

#### Step 1 — Deployment suppresses council initialization failures

The loop logs an initialization failure and continues, leaving an incompletely initialized electorate.

Source: `scripts/deploy.ts:449-493` (commit 0c74a04).

```typescript
  for (const addr of initialCouncilWallets) {
    const alreadyWhitelisted = await [project]CouncilToken.isWhitelisted(addr);
    const currentBalance = await [project]CouncilToken.balanceOf(addr);
    console.log(
      `🔎 Council wallet preflight ${addr}: whitelisted=${alreadyWhitelisted}, balance=${currentBalance.toString()}`
    );

    if (alreadyWhitelisted || currentBalance > 0n) {
      console.log(`ℹ️ Skipping council wallet already initialized: ${addr}`);
      continue;
    }

    try {
      const populated = await [project]CouncilTokenAsDeployer.whitelistAddress.populateTransaction(
        addr
      );
      const txData = populated.data;
      if (!txData || txData === "0x") {
        throw new Error(
          `whitelistAddress calldata is empty for ${addr}; refusing to send invalid tx`
        );
      }

      // Preflight with eth_call to surface revert reasons before broadcasting.
      await [project]CouncilTokenAsDeployer.whitelistAddress.staticCall(addr);

      const estimatedGas = await ethers.provider.estimateGas({
        from: deployerAddress,
        to: [project]CouncilTokenAddress,
        data: txData,
      });

      const tx = await deployer.sendTransaction({
        to: [project]CouncilTokenAddress,
        data: txData,
        gasLimit: (estimatedGas * 12n) / 10n,
      });
      await tx.wait();
      console.log(`✅ Whitelisted and minted YCGT to: ${addr}`);
    } catch (err) {
      console.warn(
        `⚠️ Failed to whitelist council wallet ${addr}: ${err instanceof Error ? err.message : String(err)}`
      );
    }
  }
```

#### Step 2 — Governance requires three usable votes

The three-vote floor is not relaxed because deployment failed. Handing off to an underfilled council can strand normal governance.

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

Historical deployment control flow and quorum requirements establish the fail-open initialization risk. No finding-specific executable original exploit test is available.

## Relevant test excerpts

No finding-specific executable test excerpt is available. The source excerpts above are the supporting evidence.
