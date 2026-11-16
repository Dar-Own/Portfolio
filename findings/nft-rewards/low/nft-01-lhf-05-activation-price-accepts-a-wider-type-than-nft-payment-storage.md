# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Low ceiling; privileged configuration denial of service  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Source/configuration analysis; no finding-specific executable exploit test  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Activation price accepts a wider type than NFT payment storage

## Description

Collection configuration accepts a nonzero uint96 activation price, while activation stores the payment as uint80. A configured value above uint80.max passes the wider configuration boundary and makes activation revert during narrowing until an authorized actor lowers it.

Prerequisites and limits: The affected range is extreme and requires an authorized configuration change. No unprivileged setter, production configuration in that range or tested deployed denial is established.

### Mechanism step by step

#### Step 1 — An administrator sets the wider price

The setter uses uint96 and rejects zero; it does not apply the narrower amountPaid ceiling.

Source: `contracts/contracts/core/[project]CollectionAdmin.sol:82-99`.

```solidity
    function setActivationPrice(
        address collection,
        uint96 price
    ) external onlyRoleOrAdmin(OPERATOR_ROLE) nonReentrant {
        Collection storage c = _collection(collection);
        _requireNotRetired(c);

        if (price == 0) revert InvalidPrice();

        if (!hasRole(DEFAULT_ADMIN_ROLE, msg.sender)) {
            bool isBandSet = c.minPrice != 0 || c.maxPrice != 0;
            if (!isBandSet || price < c.minPrice || price > c.maxPrice) revert PriceOutsideBand();
        }

        uint96 oldPrice = c.activationPrice;
        c.activationPrice = price;

        emit ActivationPriceSet(collection, oldPrice, price);
```

#### Step 2 — A holder pays the configured activation price

The activation path converts the configured price to uint80 for per-NFT storage; SafeCast rejects values outside the range.

Source: `contracts/contracts/core/[project]Activation.sol:42-52`.

```solidity
        uint96 price = c.activationPrice;
        uint256 expected = uint256(price) * n;
        if (msg.value != expected) revert WrongFunding(expected, msg.value);

        uint256 newCount = uint256(c.activatedCount) + n;
        uint32 cap = c.maxActivations;
        if (cap != 0 && newCount > cap) revert MaxActivationsReached();

        uint40 nowTs = SafeCast.toUint40(block.timestamp);
        uint32 batchId = v.openBatchId;
        uint80 paidPerToken = SafeCast.toUint80(price);
```

## Proof of concept

The accepted uint96 configuration domain exceeds the uint80 payment-storage domain. Source types and narrowing behavior establish the conditional configuration failure; no finding-specific executable test is supplied.

**Proof limits:** No finding-specific executable test; source/configuration evidence with stated prerequisites.

## Relevant test excerpts

No executable test is included for this case. The source excerpts above support the stated behavior and its prerequisites.
