# Source-code-based

Project: Anonymous reward issuance and governance protocol (RG-01)
Category: Reward issuance and governance
Severity: Low (final report assignment)
Engagement: Private engagement
Chain: EVM; Sepolia release attestation is separate from the historical local PoCs
Tech stack: Solidity, OpenZeppelin, ERC-20, Merkle proofs, Governor/Timelock, Trustline SDK, Hardhat, TypeScript
Evidence: Original local test source; mechanism assertions shown below
Remediation: Acknowledged — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Delegated backend is screened

## Description

Authorized backend submits delegated claim for clean user → SDK screens both sender and recipient → backend address blocked → delegated claim reverts despite clean recipient.

Backend claim liveness loss; clean user self-claim remains possible.

Acknowledged at commit 1c2c2fd. Engineer-facing comments on both delegated claim paths state that Trustline screens msg.sender and the recipient list at contracts/NMRP1.sol:227-234 and contracts/[project]ContributorAllocations.sol:128-135. The liveness dependency and need for backup compliant sender wallets are documented in the operational guidance. The amendment documents the behavior; it intentionally does not remove the backend sender from Trustline screening.

### Mechanism step by step

#### Step 1 — the misleading in-code comment

Authorized backend submits delegated claim for clean user

Source: `contracts/mintUscyRewards.sol:192-195` (commit 95fc21c).

```solidity
        // NEW: Trustline-validate the *user* (recipient), not msg.sender
        address[] memory addrs = new address[](1);
        addrs[0] = user;
        requireTrustline(addrs);
```

#### Step 2 — SDK requireTrustline forwards msg.sender to the engine

msg.sender (the backend on delegated claims) IS screened, contradicting the comment.

Source: `node_modules/@trustline.id/evmsdk/contracts/Trustlined.sol:103-107` (dependency source snapshot; not Git-versioned or independently pinned to audited commit `95fc21c`).

```solidity
    /// @notice Requires a trusted transaction and non‑sanctioned msg.sender + addresses[]
    /// @param addresses An array of addresses that will be verified by the policy
    function requireTrustline(address[] memory addresses) internal {
        validationEngine.requireTrustline(msg.sender, msg.value, msg.data, addresses);
    }
```

The SDK snapshot illustrates this code path; the historical applicability is supported by the final report and the original local test. This unversioned dependency excerpt does not establish the exact dependency bytes used at commit `95fc21c`.

## Proof of concept

The configurable validation mock blocks the backend sender. A delegated claim reverts while the clean recipient’s self-claim succeeds. The effect is backend-path availability loss; the test does not bypass screening or transfer funds to an attacker.

The tests deploy fresh contracts with synthetic actors. Accounting and governance fixtures use an always-allowing validation mock; M9/M10 use a configurable mock. An EOA admitted as a pool stands in for a governance-approved pool contract where the boundary is tested directly. These are local tests, despite the original configuration requiring a Sepolia fork.

Test references:

- `PoCs/helpers/deploy.ts`
- `PoCs/trustline-liveness/M10_backend_sender_screened.test.ts`
- `PoCs/trustline-liveness/M10_backend_sender_screened.test.ts:37-67`

## Relevant test excerpts

Source: `PoCs/trustline-liveness/M10_backend_sender_screened.test.ts:37-67`.

```typescript
  it("reverts delegated claim even when recipient is clean", async () => {
    const pool = await rewards.getAddress();

    await expect(
      rewards
        .connect(core.authorizedParty)
        .claimRewards(
          user,
          [uuid],
          [amount],
          [built.accountIDHashes[0]],
          [built.proofs[0]],
          [pool],
          [1]
        )
    ).to.be.revertedWith("sender blocked");

    await rewards
      .connect(core.attacker)
      .claimRewards(
        user,
        [uuid],
        [amount],
        [built.accountIDHashes[0]],
        [built.proofs[0]],
        [pool],
        [1]
      );
    expect(await core.[project].balanceOf(user)).to.equal(amount);
  });
});
```
