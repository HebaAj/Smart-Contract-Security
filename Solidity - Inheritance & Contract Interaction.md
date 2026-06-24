---
tags: [solidity, smart-contracts]
---

One-line summary: composing contracts via inheritance, talking to other deployed contracts via interfaces, and the `Ownable` access-control pattern that ties both together.

---

## Inheritance

`contract B is A` gives `B` access to everything `A` declares as `public` or `internal` (not `private`).

```solidity
contract Ownable {
    address public owner;
    constructor() { owner = msg.sender; }

    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }
}

contract Game is Ownable {
    // inherits `owner` and `onlyOwner` — no need to redeclare
    function setDifficulty(uint _d) external onlyOwner {
        difficulty = _d;
    }
}
```

Used both for genuine is-a relationships and just for organizing shared logic across files (`import "./Ownable.sol";`).

---

## `Ownable` Pattern

The example above *is* the pattern — OpenZeppelin's `Ownable` is the audited, standard version of exactly this (plus `transferOwnership` / `renounceOwnership`). Worth pulling from a vetted library rather than rewriting, for the same reason you don't roll your own crypto: small mistakes in access control are exactly where exploits live.

> A contract being on Ethereum doesn't make it decentralized. `onlyOwner` functions are a **trust assumption** — read what they can actually do (change fees? drain funds? swap out a linked contract address?) before treating "it's a smart contract" as a safety guarantee. This is the first thing to check when auditing.

---

## `msg.sender` + `require`

`msg.sender` — the address that called the current function — is the only identity primitive that matters for access control, and it's backed by the chain's cryptography: spoofing it requires the caller's private key.

```solidity
mapping (address => uint) public balances;

function withdraw(uint _amount) external {
    require(balances[msg.sender] >= _amount); // reverts everything if false
    balances[msg.sender] -= _amount;
    payable(msg.sender).transfer(_amount);
}
```

`require(condition)` reverts the entire transaction (all state changes undone) if `condition` is false — the standard guard-clause for preconditions, access control, and input validation.

---

## Addresses

20 bytes, uniquely identifies an account — either an externally-owned account (controlled by a private key) or a contract. From the EVM's perspective, both look the same: an address you can send a message (or Ether) to.

---

## Interfaces

An interface is a **calling-convention declaration**, not a contract you deploy:

```solidity
interface IGameToken {
    function balanceOf(address _owner) external view returns (uint);
}
```

- Only signatures, no bodies (`;` instead of `{}`).
- Never deployed — erased after compilation.

```solidity
contract Game {
    IGameToken tokenContract;

    constructor(address _tokenAddress) {
        tokenContract = IGameToken(_tokenAddress); // typecast, NOT instantiation
    }

    function checkBalance(address _player) external view returns (uint) {
        return tokenContract.balanceOf(_player); // generates an external CALL
    }
}
```

`IGameToken(_tokenAddress)` doesn't create anything — it tells the *compiler* "treat this address as something with `balanceOf`'s signature," so it can compute the right 4-byte function selector and ABI-encode the arguments for the call. The real contract at `_tokenAddress` was deployed independently and may have more functions than the interface declares — the interface only needs to cover what you call.

> The interface is an *assumption*. If `_tokenAddress` doesn't actually implement `balanceOf(address)` with that signature, the call either reverts or — if some other function happens to share that selector — silently calls the wrong function. Verify the address really is what the interface claims, especially if it's settable (see immutability below).

---

## Immutability vs. Upgradeability

Deployed code is permanent — no patching a live bug. Two consequences pull in opposite directions:

- **Trust**: "code is law" — verified behavior can't be changed out from under users.
- **Risk**: a bug is permanent too, unless the contract was designed with an escape hatch (e.g. a settable external-contract address, gated by `onlyOwner`).

```solidity
address public tokenContractAddress;

function setTokenContract(address _new) external onlyOwner {
    tokenContractAddress = _new; // the escape hatch
}
```

Every escape hatch is also a centralization point — it's the same `onlyOwner` trust assumption from above, just applied to "can redirect where this contract gets its data/logic from." Worth weighing deliberately, not defaulting to either extreme.

---

**See also:** [Solidity - Functions, Visibility & Modifiers](./Solidity%20-%20Functions%2C%20Visibility%20%26%20Modifiers.md) · [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
