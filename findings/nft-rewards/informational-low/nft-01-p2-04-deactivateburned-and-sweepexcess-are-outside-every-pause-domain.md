# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Informational to Low ceiling; pause-matrix design decision  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Source/configuration analysis; no finding-specific executable exploit test  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# `deactivateBurned` and `sweepExcess` are outside every pause domain

## Description

deactivateBurned and sweepExcess are default-admin operations outside the activation/claim/reporting pause domains. Pausing user activity therefore does not stop these administrator lifecycle and excess-fund operations.

Prerequisites and limits: Both functions are explicitly privileged, and unpausable containment powers may be intentional. No authorization bypass or unauthorized withdrawal is shown. The open issue is whether the documented incident-response policy should permit these operations.

### Mechanism step by step

#### Step 1 — An administrator calls deactivation during a pause

The entry point applies role and reentrancy checks but no domain-pause check.

Source: `contracts/contracts/core/[project]Deactivation.sol:14-25`.

```solidity
    function deactivateBurned(address collection, uint256[] calldata tokenIds)
        external
        onlyRole(DEFAULT_ADMIN_ROLE)
        nonReentrant
    {
        CoreStorage storage $ = _getCoreStorage();

        Collection storage c = _collection(collection);

        _requireNotRoundDue(c.vaultId);

        uint256 n = tokenIds.length;
```

#### Step 2 — An administrator can still sweep excess

sweepExcess checks admin authority and available excess but is not pause-gated.

Source: `contracts/contracts/core/[project]VaultAdmin.sol:194-206`.

```solidity
    function sweepExcess(address to, uint256 amount) external onlyRole(DEFAULT_ADMIN_ROLE) nonReentrant {
        if (to == address(0)) revert InvalidWallet();

        if (amount == 0) revert ZeroAmount();

        uint256 balance = address(this).balance;
        uint256 liability = _getCoreStorage().totalLiability;
        if (amount > balance || balance - amount < liability) revert ExceedsLiability();
        uint256 remainingExcess = balance - amount - liability;

        (bool ok,) = to.call{value: amount}("");
        if (!ok) revert PayoutFailed(to, amount);

```

## Proof of concept

Source establishes that two administrator operations omit the domain pause guards. This may be deliberate containment policy; no unauthorized action or executable exploit is demonstrated.

**Proof limits:** No finding-specific executable test; source/configuration evidence with stated prerequisites.

## Relevant test excerpts

No executable test is included for this case. The source excerpts above support the stated behavior and its prerequisites.
