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

# Trustline gate ships fail-open / never enabled

## Description

Deployer uses zero Trustline proxy → fresh validation engine starts sanctions disabled → deployment omits enablement → apparently screened issuance operates with sanctions check off.

Original compliance gate fails open under the documented fresh-engine configuration.

Resolved at commits 6be9a56 and 9ed1b41. assertTrustlineConfiguration fails closed unless sanctionsOracleEnabled() is true and trustlineOracleEnabled() is false; when an approved sanctions oracle is supplied, it also requires an exact live-address match at scripts/trustlineGuards.ts:376-428. The mainnet deployment script requires the approved oracle and makes the sanctions-only check mandatory at scripts/deploy.ts:73-83 and lines 1082-1094. The completed Sepolia verification supplied the approved oracle, verified [project] compliance enforcement, and recorded the observed state at scripts/verify-trustline-config.ts:269-301 and lines 358-415.

### Mechanism step by step

#### Step 1 — proxy hardcoded to zero, passed to both pools

trustlineValidationEngineProxy = ZeroAddress forces the SDK deploy branch.

Source: `scripts/deploy.ts:99-101` (commit 95fc21c).

```typescript
  const validationEngine = process.env.TRUSTLINE_VAL_ENGINE;
  if (!validationEngine) throw new Error("Missing TRUSTLINE_VAL_ENGINE env var");
  const trustlineValidationEngineProxy = ethers.ZeroAddress;
```

Source: `scripts/deploy.ts:144-145` (commit 95fc21c).

```typescript
    validationEngine,
    trustlineValidationEngineProxy
```

Source: `scripts/deploy.ts:161-162` (commit 95fc21c).

```typescript
    validationEngine,
    trustlineValidationEngineProxy
```

#### Step 2 — SDK takes the deploy branch when proxy==0

proxy==0 -> new ERC1967Proxy(logic, initialize(msg.sender)). a brand-new per-pool engine.

Source: `node_modules/@trustline.id/evmsdk/contracts/Trustlined.sol:72-86` (dependency source snapshot; not Git-versioned or independently pinned to audited commit `95fc21c`).

```solidity
        } else {
            // Deploy a new Validation Engine proxy and initialize it atomically (never deploy manually).
            // Proxy address uses CREATE and is chain-/nonce-dependent — not reproducible cross-chain.
            _assertContractAccount(logic);

            address initialOwner = msg.sender;

            // Deployment of the Validation Engine proxy
            bytes memory data = abi.encodeCall(IValidationEngineInitializer.initialize, (initialOwner));
            address proxy_ = address(new ERC1967Proxy(logic, data));

            _assertValidationEngine(proxy_);
            validationEngine = IValidationEngine(proxy_);

            emit ValidationEngineDeployed(address(this), proxy_, logic, initialOwner);
```

The fresh-engine disabled-screening default is attributed by the final report to the separate dependency audit. Repository deployment code alone proves the zero-proxy branch and missing enablement, not an independently verified historical engine default or live operational configuration.

The SDK snapshot illustrates this code path; the historical applicability is supported by the final report. This unversioned dependency excerpt does not establish the exact dependency bytes used at commit `95fc21c`.

## Proof of concept

Deployment and dependency configuration establish the fail-open initialization path. No finding-specific executable exploit test is available. The disabled-screening default is attributed to the dependency assessment, not independently established by the displayed repository code alone.

## Relevant test excerpts

No finding-specific executable test excerpt is available. The source excerpts above are the supporting evidence.
