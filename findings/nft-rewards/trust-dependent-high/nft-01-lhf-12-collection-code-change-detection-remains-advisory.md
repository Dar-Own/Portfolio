# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Trust-dependent High ceiling only with untrusted collection upgrade authority; no High vulnerability established  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Source/configuration analysis; no finding-specific executable exploit test  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Collection code-change detection remains advisory

## Description

The protocol records collection code hash and exposes a codeChanged indicator, but ownership-sensitive actions continue calling the configured collection’s current code. A hostile mutable collection could change ownerOf semantics, affecting claim authority or creating unreadability and false-burn conditions.

Prerequisites and limits: The November 4 census confirms external-code dependencies and three proxy collections sharing an admin contract. Four additional fixed clones also execute external implementation code, but fixed clones are not proof of upgradeability. The owner/control policy of the shared admin and hostile upgrades are not demonstrated. Every recorded code_changed value was false at the pin. Proxy implementation changes can also occur without changing proxy runtime code hash; no deployed exploit is claimed.

### Mechanism step by step

#### Step 1 — Onboarding records a code-hash observation

The collection configuration captures codeHashAtAdd as metadata.

Source: `contracts/contracts/core/[project]CollectionAdmin.sol:23-48`.

```solidity
        if (nft.code.length == 0 || !ERC165Checker.supportsInterface(nft, ERC721_INTERFACE_ID)) {
            revert UnsupportedStandard(nft);
        }

        Vault storage v = _vault(vaultId);

        if (v.collectionCount >= $.maxCollectionsPerVault) revert VaultFull();

        _requireNotShort(v);

        if (price == 0) revert InvalidPrice();

        uint32 gasAllowance = ownerOfGas == 0 ? DEFAULT_OWNER_OF_GAS : ownerOfGas;
        uint40 memberSince = _nextGridCutoff(v.roundAnchor, v.roundInterval, uint40(block.timestamp));
        uint32 index = uint32($.collectionList.length);
        bytes32 codeHash = nft.codehash;

        c.vaultId = vaultId;
        c.collectionIndex = index;
        c.status = uint8(Status.Active);
        c.activationEnabled = activationEnabled;
        c.memberSince = memberSince;
        c.activationPrice = price;
        c.maxActivations = maxActivations;
        c.ownerOfGas = gasAllowance;
        c.codeHashAtAdd = codeHash;
```

#### Step 2 — The dashboard reports a difference

The read path computes codeChanged; it does not enforce an execution block.

Source: `contracts/contracts/lens/LensDashboards.sol:67-70`.

```solidity

        page.shareE9 = _shareE9(c.xphar, v.xpharTotal);

        page.codeChanged = collection.codehash != c.codeHashAtAdd;
```

#### Step 3 — Ownership-sensitive code uses the current collection implementation

The bounded ownerOf call delegates authority semantics to whatever code the collection executes at the time.

Source: `contracts/contracts/core/[project]Storage.sol:211-243`.

```solidity
    function _ownerOf(address collection, uint256 tokenId) internal view returns (address owner) {
        uint256 stipend = _getCoreStorage().collections[collection].ownerOfGas;
        bytes memory callData = abi.encodeWithSelector(IERC721.ownerOf.selector, tokenId);
        bool wellFormed;
        assembly ("memory-safe") {
            let success := staticcall(stipend, collection, add(callData, 0x20), mload(callData), 0, 0)
            if and(success, eq(returndatasize(), 32)) {
                returndatacopy(0, 0, 32)
                let word := mload(0)
                if iszero(shr(160, word)) {
                    wellFormed := 1
                    owner := word
                }
            }
        }
        if (!wellFormed) revert TokenUnreachable(collection, tokenId);
    }

    function _tryOwnerOf(address collection, uint256 tokenId) internal view returns (bool burned) {
        uint256 stipend = _getCoreStorage().collections[collection].ownerOfGas;
        bytes memory callData = abi.encodeWithSelector(IERC721.ownerOf.selector, tokenId);
        bool live;
        assembly ("memory-safe") {
            let success := staticcall(stipend, collection, add(callData, 0x20), mload(callData), 0, 0)
            if and(success, eq(returndatasize(), 32)) {
                returndatacopy(0, 0, 32)
                let word := mload(0)
                if and(iszero(shr(160, word)), iszero(iszero(word))) {
                    live := 1
                }
            }
        }
        burned = !live;
```

## Proof of concept

A historical Avalanche census at block 99430359 recorded 12 collections: three EIP-1967 proxies sharing one admin, two EIP-1167 clones, two fixed minimal clones, and five immutable-appearing contracts. Seven execute external implementation code, but only the three proxies establish upgradeability through the shared authority. Every code_changed value was false. The census is configuration evidence, not an exploit test or proof of hostile control.

**Proof limits:** Trust surface only; malicious upgrades and control of the shared administrator are not demonstrated.

## Relevant test excerpts

No executable test is included for this case. The source excerpts above support the stated behavior and its prerequisites.
