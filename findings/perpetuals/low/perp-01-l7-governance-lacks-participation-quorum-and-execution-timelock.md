# Source-code-based

Project: Name-redacted perpetuals protocol (PERP-01)
Category: Perpetuals
Severity: Low — final client-report assignment; historical finding
Engagement: Private engagement
Chain: Solana
Tech stack: Rust, Anchor, Solana runtime, SPL Token/SPL Governance, oracle integrations; Rust local tests and configured Surfpool evidence where stated
Evidence: Source/configuration analysis
Remediation: Acknowledged — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Governance Lacks Participation Quorum and Execution Timelock

## Description

A proposer obtains the configured 45% approval threshold → governance has no separate minimum-participation quorum and transactions_hold_up_time=0 → an approved proposal can execute immediately. The report describes the threshold denominator as deposited votes, not votes cast.

Low-participation governance takeover opportunity without reaction delay; historical 272-proposal study shows dominant bloc participation, not an actual takeover.

No executable governance-takeover PoC or raw independent replay receipt was established. The final report summarizes 272 historical proposals and 210 passing proposals, dominated by one voting bloc. This is historical configuration/participation analysis; it does not demonstrate a takeover. Inactivity does not itself reduce a fixed deposited-weight threshold. The claim that any tiny bloc could pass should not be inferred from absence of a separate quorum.

The excerpts span the affected implementation and later remediation where identified. They do not assert equivalence to a current deployment.

### Mechanism step by step

#### Step 1 — Approve under the configured percentage threshold

The client configuration uses a 45% YesVotePercentage threshold. The final report describes deposited-vote weight as the denominator, not 45% of votes cast.

Source: `cli/src/client.ts:1487-1496`

```typescript
    const cfg = new splGov.GovernanceConfig({
      minCommunityTokensToCreateProposal: minTokensToCreateProposal,
      minCouncilTokensToCreateProposal: new BN(0),
      communityVoteThreshold: {
        type: splGov.VoteThresholdType.YesVotePercentage,
        value: 45,
      },
      minInstructionHoldUpTime: 0,
      baseVotingTime: 3 * (3600 * 24),
      communityVoteTipping: splGov.VoteTipping.Strict,
```

#### Step 2 — Execute without a separate holding interval or veto

The same configuration sets zero hold-up/cool-off and disables council/veto thresholds. The final report attributes the finding to historical SPL Governance account state, not a missing check in the perpetuals program.

Source: `cli/src/client.ts:1497-1512`

```typescript
      councilVoteThreshold: {
        type: splGov.VoteThresholdType.Disabled,
        value: undefined,
      },
      councilVetoVoteThreshold: {
        type: splGov.VoteThresholdType.Disabled,
        value: undefined,
      },
      communityVetoVoteThreshold: {
        type: splGov.VoteThresholdType.Disabled,
        value: undefined,
      },
      councilVoteTipping: splGov.VoteTipping.Disabled,
      votingCoolOffTime: 0,
      depositExemptProposalCount: 0,
    });
```

## Proof of concept

No executable exploit test is available for this case. The evidence is source/configuration inspection and the historical assessment.

No executable governance-takeover PoC or raw independent replay receipt was established. The final report summarizes 272 historical proposals and 210 passing proposals, dominated by one voting bloc. This is historical configuration/participation analysis; it does not demonstrate a takeover. Inactivity does not itself reduce a fixed deposited-weight threshold. The claim that any tiny bloc could pass should not be inferred from absence of a separate quorum.

## Relevant test excerpts

No finding-specific executable test excerpt is available. The source excerpts above are the supporting evidence.
