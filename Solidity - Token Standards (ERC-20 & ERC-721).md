---
tags: [solidity, smart-contracts, tokens, security]
---

How the ERC-20 and ERC-721 standards define fungible and non-fungible tokens, and why conforming to a standard beats rolling your own.

---

## Why Standards Exist

A token is just a contract with a balance ledger and transfer logic. The problem: if every project invents its own interface, every wallet, exchange, and app has to write custom integration code per token.

ERC standards solve this by defining a shared interface. Any contract implementing ERC-20 can be listed on any DEX, held in any wallet, and composed into any protocol — without those systems knowing anything specific about the token.

> Standards are composability infrastructure. An audited exchange doesn't need to trust your token logic — it just needs to know you implement `transferFrom`.

---

## ERC-20 — Fungible Tokens

Every unit is identical and interchangeable (like currency).

Core interface:

```solidity
// Any two units of the same ERC-20 are identical — no token ID
function balanceOf(address _owner) external view returns (uint256);
function transfer(address _to, uint256 _amount) external returns (bool);
function transferFrom(address _from, address _to, uint256 _amount) external returns (bool);
function approve(address _spender, uint256 _amount) external returns (bool);
function allowance(address _owner, address _spender) external view returns (uint256);
```

Internal state is typically just:

```solidity
mapping(address => uint256) balances;
mapping(address => mapping(address => uint256)) allowances; // owner => spender => amount
```

`approve` + `transferFrom` is the two-step delegation pattern: owner pre-authorizes a spender (e.g. a DEX contract) to pull up to a certain amount. The DEX calls `transferFrom` on behalf of the user without ever holding the tokens itself.

> Auditor flag: ERC-20 `approve` has a known race condition — if an owner changes an allowance from X to Y, a spender can front-run and spend X before the update, then spend Y after. The fix is `increaseAllowance`/`decreaseAllowance` (OpenZeppelin pattern) instead of direct `approve`.

---

## Partial Restrictions: Overriding One Path, Not All

`transfer` and `transferFrom` are two **independent** entry points into the same underlying balance change — overriding one to add a restriction does nothing to the other unless both are overridden.

```solidity
contract TimeLockedToken is ERC20 {
    uint256 public unlockTime;
    address public founder;

    constructor(address _founder) ERC20("Locked", "LCK") {
        founder = _founder;
        unlockTime = block.timestamp + 365 days;
        _mint(founder, 1_000_000 * 10 ** decimals());
    }

    // Only this path is restricted...
    function transfer(address _to, uint256 _value) public override returns (bool) {
        if (msg.sender == founder) {
            require(block.timestamp > unlockTime, "still locked");
        }
        return super.transfer(_to, _value);
    }

    // ...transferFrom is inherited unmodified from ERC20 — no time check at all
}
```

The founder can't call `transfer` early, but nothing stops them from calling `approve(founder, balance)` followed by `transferFrom(founder, founder, balance)` (or any other recipient) — `transferFrom` was never touched, so it has no lock. The two functions move the same balances mapping; restricting the name of the function someone calls does not restrict the underlying state change.

> Auditor flag: any contract that overrides `transfer` for an access-control or time-lock check without applying the identical check to `transferFrom` (or vice versa). This applies symmetrically — locking only `transferFrom` while leaving `transfer` open is the same bug. The fix is a shared internal function (or modifier) both public functions route through.

---

## ERC-721 — Non-Fungible Tokens (NFTs)

Every token has a unique ID and is not interchangeable. Ownership is tracked per token, not per address balance.

Core interface:

```solidity
// Each token is unique — tracked by tokenId, not amount
function balanceOf(address _owner) external view returns (uint256);       // how many NFTs owner holds
function ownerOf(uint256 _tokenId) external view returns (address);       // who owns this specific token
function transferFrom(address _from, address _to, uint256 _tokenId) external payable;
function approve(address _approved, uint256 _tokenId) external payable;   // approve one specific token
```

Internal state:

```solidity
mapping(uint256 => address) tokenOwner;      // tokenId → owner
mapping(address => uint256) ownerTokenCount; // owner → how many tokens they hold
mapping(uint256 => address) tokenApprovals;  // tokenId → approved address
```

---

## Two Transfer Paths in ERC-721

**Path 1 — Direct transfer:** owner calls `transferFrom` themselves.

```solidity
// Owner sends their own token directly
nft.transferFrom(msg.sender, recipient, tokenId);
```

**Path 2 — Delegated transfer:** owner first calls `approve` to authorize another address, then that address (or the owner) calls `transferFrom`.

```solidity
// Step 1: owner pre-authorizes a marketplace contract for one token
nft.approve(marketplaceAddress, tokenId);

// Step 2: marketplace pulls the token when a sale completes
// (called from within the marketplace contract)
nft.transferFrom(seller, buyer, tokenId);
```

This is how NFT marketplaces work without ever taking custody of your token.

> Zero-address guard is mandatory on both paths. Transferring to `address(0)` burns the token permanently — no private key, no recovery. Always: `require(_to != address(0), "transfer to zero address")`.

---

## ERC-20 vs ERC-721 — Quick Reference

| Property | ERC-20 | ERC-721 |
|---|---|---|
| Units | Fungible (identical) | Non-fungible (unique IDs) |
| Divisible | Yes (`uint256` amounts) | No (whole tokens only) |
| Tracks | Balance per address | Owner per tokenId |
| `approve` scope | Amount allowance | Single token |
| Use case | Currency, governance tokens | Collectibles, deeds, credentials |

---

## Inheriting a Standard

Import the interface and inherit — the compiler then enforces that you implement every required function:

```solidity
import "./IERC721.sol";

contract AssetRegistry is IERC721 {
    // compiler error if any interface function is missing
    function balanceOf(address _owner) external view override returns (uint256) { ... }
    function ownerOf(uint256 _tokenId) external view override returns (address) { ... }
    // ...
}
```

> Rolling your own transfer logic instead of conforming to a standard means marketplaces and wallets can't interact with your tokens without custom integration. It also means you miss battle-tested security patterns from OpenZeppelin's reference implementations.

---

**See also:** [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) · [Solidity - Inheritance & Contract Interaction](./Solidity%20-%20Inheritance%20%26%20Contract%20Interaction.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
