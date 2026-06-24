---
tags: [solidity, smart-contracts]
---

One-line summary: who can call a function, what it's allowed to touch, and how modifiers factor out repeated checks.

---

## Visibility

| Modifier | Callable from outside contract? | Callable from other functions in same contract? | Inherited by subclasses? |
|---|---|---|---|
| `public` (default) | yes | yes | yes |
| `external` | yes | not directly (needs `this.f()`, which costs more) | yes |
| `internal` | no | yes | yes |
| `private` | no | yes | no |

Practical defaults: mark anything not meant for outside callers `private` (or `internal` if subclasses need it), and only make `public`/`external` what the contract is meant to expose. Every `public`/`external` function is part of the contract's attack surface.

`external` vs `public` for functions only ever called from outside: `external` is slightly cheaper, because `public` function arguments get copied to memory regardless of caller, while `external` can read directly from `calldata`.

---

## State Mutability

| Modifier | Reads state? | Writes state? | Gas when called externally |
|---|---|---|---|
| (none) | yes | yes | costs gas — creates a tx |
| `view` | yes | no | **free**, if called externally |
| `pure` | no | no | **free**, if called externally |

```solidity
function totalPlayers() external view returns (uint) {
    return players.length;          // reads state, changes nothing
}

function _square(uint x) private pure returns (uint) {
    return x * x;                   // depends only on its arguments
}
```

> "Free if called externally" is the key qualifier. If a non-`view` function calls a `view` function **internally**, the `view` function's work is just part of the enclosing transaction — it still costs gas, because the whole transaction still needs to be verified by every node. `view`/`pure` only skip the transaction entirely when they're the call's entry point.

---

## Constructors

Runs exactly once, at deployment — the standard place to set state like the owner.

```solidity
address public owner;

constructor() {
    owner = msg.sender; // whoever deploys becomes the owner
}
```

---

## Function Modifiers

A modifier wraps a function body. `_;` marks where that body gets substituted in.

```solidity
modifier onlyOwner() {
    require(msg.sender == owner);
    _;
}

function setFee(uint _fee) external onlyOwner {
    fee = _fee;
}
// equivalent, written out, to:
// function setFee(uint _fee) external {
//     require(msg.sender == owner);
//     fee = _fee;
// }
```

The win isn't capability — an inline `require` does the same thing. It's that the check exists in **one place**, and a contract's access-control surface becomes scannable from function *signatures* alone, without reading every body. For an auditor, that's the difference between "check one definition" and "check N call sites are all consistent."

### Modifiers with Arguments

Modifiers take parameters just like functions, and the decorated function passes them through:

```solidity
modifier aboveLevel(uint32 _level, uint _id) {
    require(idToPlayer[_id].level >= _level);
    _;
}

function enterArena(uint _id) external aboveLevel(10, _id) {
    // only reachable if idToPlayer[_id].level >= 10
}
```

### `_;` Placement

`_;` doesn't have to be last — code after it runs *after* the wrapped function body, e.g. for refunding excess payment or post-condition checks.

---

## Events

A log entry, not a return value — the mechanism for notifying off-chain listeners (web3.js/ethers.js front-ends) that something happened.

```solidity
event PlayerLeveledUp(uint indexed playerId, uint32 newLevel);

function levelUp(uint _id) external {
    idToPlayer[_id].level += 1;
    emit PlayerLeveledUp(_id, idToPlayer[_id].level);
}
```

Front-end listens for `PlayerLeveledUp` events. Emitting costs gas (events are stored in transaction logs) but far less than an equivalent storage write.

---

**See also:** [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md) · [Solidity - Inheritance & Contract Interaction](./Solidity%20-%20Inheritance%20%26%20Contract%20Interaction.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
