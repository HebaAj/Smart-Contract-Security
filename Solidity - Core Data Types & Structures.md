---
tags: [solidity, smart-contracts]
---

One-line summary: value types, structs, arrays, mappings, and how time is represented — the building blocks every contract is assembled from.

---

## Value Types: the `uint` Family

`uint` is shorthand for `uint256`. Smaller variants (`uint8`, `uint16`, ... `uint248`) exist, but as standalone state variables they reserve the same 256-bit storage slot — no gas saved. The only place sub-types pay off is inside structs (packing — see [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md)).

```solidity
uint256 totalPlayers; // == uint totalPlayers
uint8 difficulty;     // same storage cost as uint256 here, on its own
```

> Default to `uint` everywhere except inside structs where packing matters.

---

## Structs

A struct groups related fields into one custom type — Solidity's closest thing to a record/object, but with no methods.

```solidity
struct Player {
    string name;
    uint256 lastActionTime;
    uint32 level;
    uint32 xp;
}
```

Field order is cosmetic for behavior but **not** for gas — see packing in [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md).

---

## Arrays

Two flavors:

```solidity
uint[5] fixedScores;     // fixed length, set at declaration
Player[] public players; // dynamic, grows with .push()
```

- `public` on an array (or mapping) auto-generates a **getter only** — no setter. `players[2]` becomes callable externally as `players(2)`.
- `.push()` appends — insertion order is preserved, which matters when you later need to find/remove a specific element (cost implications in [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md)).

---

## Mappings

A hash table: key → value, with **zero iteration and no `.length`**. Any key you haven't written returns the type's zero-value (`0`, `""`, `false`, a zeroed struct).

```solidity
mapping (address => uint) public playerToId;
mapping (uint => Player) public idToPlayer;
```

> No way to enumerate keys or check "does this key exist" directly — you either track existence with a separate flag/field, or rely on the zero-value convention (e.g. `id == 0` means "unset", as long as `0` is never a valid id).

---

## Strings

UTF-8, arbitrary length, reference type → needs a data-location keyword when used as a function parameter (`memory` / `calldata` / `storage`, see [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md)).

Solidity has **no `==` for strings**. Equality is done via hashing — see [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md).

---

## Time

`block.timestamp` (the modern name — `now` was an alias for the same thing, removed in Solidity ≥0.7.0) returns the current block's Unix timestamp as `uint256`.

```solidity
uint32 constant COOLDOWN = 1 days; // time-unit literals convert to seconds: 86400

function act(uint _id) external {
    require(block.timestamp >= idToPlayer[_id].lastActionTime + COOLDOWN);
    idToPlayer[_id].lastActionTime = uint32(block.timestamp);
}
```

> Storing timestamps as `uint32` instead of `uint256` saves storage (relevant inside structs — packing) but overflows in 2106. A real trade-off, not a toy detail — pick deliberately based on the contract's expected lifetime.

---

**See also:** [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md) · [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
