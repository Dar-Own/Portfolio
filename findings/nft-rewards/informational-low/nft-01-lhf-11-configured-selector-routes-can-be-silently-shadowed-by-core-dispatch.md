# Source-code-based

Project: Anonymous NFT rewards and off-chain conversion protocol (NFT-01)  
Category: NFT rewards  
Severity: Informational to Low ceiling; upgrade/hotfix configuration footgun  
Engagement: Private engagement (research-stage review)  
Chain: Avalanche C-Chain; deployment/runtime equivalence has not been established  
Tech stack: Solidity 0.8.36, EVM Cancun, OpenZeppelin 5.6.1, ERC-721, UUPS/ERC-1967, ERC-7201 storage, delegatecall modules, Hardhat3/ethers6; TypeScript/Node.js, PostgreSQL indexer, React19/Vite app  
Evidence: Source/configuration analysis; no finding-specific executable exploit test  
Remediation: Not independently verified  
Contribution: Team-engagement case study; no sole-discovery claim  
Classification: Research candidate / hardening observation; severity is a ceiling, not an accepted vulnerability rating  
Proof availability: Excerpts and source references; the original harness and dependencies are not distributed.

# Configured selector routes can be silently shadowed by Core dispatch

## Description

An upgrader can register a module route for a selector implemented directly by Core. The route mapping can report the module while EVM dispatch continues executing the direct Core function, making the configured hotfix inert.

Prerequisites and limits: Requires authorized router configuration. No unauthorized upgrade, deployed inert hotfix or executed route-shadow PoC is established. It is a source dispatch/property observation.

### Mechanism step by step

#### Step 1 — An upgrader registers a selector route

The routing setter records selector-to-module assignments after module validation.

Source: `contracts/contracts/core/[project]ModuleRouter.sol:20-45`.

```solidity
    function setSelectorModule(bytes4[] calldata selectors, address module)
        external
        onlyRole(UPGRADER_ROLE)
        nonReentrant
    {
        uint256 count = selectors.length;
        if (count == 0) revert EmptyList();

        if (module != address(0)) _requireValidModule(module);

        RouterStorage storage r = _getRouterStorage();
        for (uint256 i; i < count; ++i) {
            bytes4 selector = selectors[i];
            address previous = r.moduleOf[selector];
            r.moduleOf[selector] = module;
            emit SelectorModuleSet(selector, previous, module);
        }
    }

    function adminModule() external view returns (address module) {
        module = _getCoreStorage().adminModule;
    }

    function selectorModule(bytes4 selector) external view returns (address module) {
        module = _getRouterStorage().moduleOf[selector];
    }
```

#### Step 2 — Only unknown selectors reach the router fallback

Routing is implemented in fallback; direct Core selectors are selected before that fallback.

Source: `contracts/contracts/core/[project]ModuleRouter.sol:63-80`.

```solidity
    fallback() external payable {
        address module = _getRouterStorage().moduleOf[msg.sig];
        if (module == address(0)) {
            if (msg.value != 0) revert ValueNotAccepted(msg.sig);
            module = _getCoreStorage().adminModule;
            if (module == address(0)) revert AdminModuleNotSet();
        }

        assembly ("memory-safe") {
            let ptr := mload(0x40)
            calldatacopy(ptr, 0, calldatasize())
            let ok := delegatecall(gas(), module, ptr, calldatasize(), 0, 0)
            let size := returndatasize()
            returndatacopy(ptr, 0, size)
            switch ok
            case 0 { revert(ptr, size) }
            default { return(ptr, size) }
        }
```

#### Step 3 — A direct Core implementation remains callable

version is a direct Core method and illustrates a selector that can shadow an entry in the route map.

Source: `contracts/contracts/[project]Core.sol:74-76`.

```solidity
    function version() external pure virtual returns (string memory v) {
        v = "1.0.0";
    }
```

## Proof of concept

Core selectors and router dispatch are compared in source. No executable route-shadow test or deployed ineffective hotfix is supplied.

**Proof limits:** No finding-specific executable test; source/configuration evidence with stated prerequisites.

## Relevant test excerpts

No executable test is included for this case. The source excerpts above support the stated behavior and its prerequisites.
