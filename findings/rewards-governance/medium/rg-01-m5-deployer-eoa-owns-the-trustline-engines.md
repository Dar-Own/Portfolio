# Source-code-based

Project: Anonymous reward issuance and governance protocol (RG-01)
Category: Reward issuance and governance
Severity: Medium (final report assignment)
Engagement: Private engagement
Chain: EVM; Sepolia release attestation is separate from the historical local PoCs
Tech stack: Solidity, OpenZeppelin, ERC-20, Merkle proofs, Governor/Timelock, Trustline SDK, Hardhat, TypeScript
Evidence: Source/configuration analysis
Remediation: Resolved — historical report disposition; current deployment state is not asserted
Contribution: Team security engagement; this case study does not claim sole discovery.
Proof availability: Excerpts and source references; the original harness is not distributed.

# Deployer EOA owns the Trustline engines

## Description

Deployer passes zero Trustline proxy → SDK instantiates per-pool engines owned by deployer → no final ownership handoff → EOA retains engine administration.

Compliance/liveness controls remain under deployer EOA; final release now bound to attested timelock handoff.

Resolved at commits a79e61b, a2fa2ad, and b486490, merged unchanged at remediation head 347946a. Deployment initiates the delayed Trustline administration transfer at scripts/deploy.ts:1036-1042. The release manifest pins the chain, proxy, minter, implementation, implementation code hash, and expected administrator at scripts/trustline-release-manifest.json:3-10. CI requires equality across that manifest, deployed_addresses.json, and the verification schedule at .github/workflows/verify-trustline.yml:135-152. The verifier then binds its RPC target and expected values to the manifest and requires a completed administrator transfer at scripts/verify-trustline-config.ts:203-281.

### Mechanism step by step

#### Step 1 — SDK sets initialOwner = msg.sender on the freshly-deployed engine

msg.sender during the pool constructor is the deploying EOA -> it owns the engine proxy.

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

#### Step 2 — deploy uses the logic+zero-proxy form (same as H1 SNIPPET 1)

SDK instantiates per-pool engines owned by deployer

Source: `scripts/deploy.ts:101-101` (commit 95fc21c).

```typescript
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

The SDK snapshot illustrates this code path; the historical applicability is supported by the final report. This unversioned dependency excerpt does not establish the exact dependency bytes used at commit `95fc21c`.

## Proof of concept

The deployment branch and validation-engine initialization show the administrator handoff boundary. No finding-specific executable original exploit test is available; later release attestation concerns remediation.

## Relevant test excerpts

No finding-specific executable test excerpt is available. The source excerpts above are the supporting evidence.
