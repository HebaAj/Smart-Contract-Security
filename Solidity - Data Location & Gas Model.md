---
tags: [solidity, smart-contracts]
---

One-line summary: storage vs memory vs calldata, what each costs, and why "more computation, less storage" is usually the right trade in Solidity.

---

## Gas, Briefly

Every operation has a fixed gas cost (defined by the EVM); a transaction's total cost = sum of its operations' costs × gas price. This exists because every node re-executes every transaction — gas is what makes infinite loops and resource-hogging computation economically (and ultimately mechanically) impossible.

The costs are wildly uneven:

| Operation | Approx. cost |
|---|---|
| Simple arithmetic (`+`, `-`) | ~3 gas |
| Memory read/write | a few gas |
| Storage read (warm) | ~100 gas |
| Storage read (cold) | ~2,100 gas |
| Storage write (existing slot) | ~5,000 gas |
| Storage write (new slot) | ~20,000 gas |

Storage dominates by 1–3 orders of magnitude. Everything below follows from that one fact.

---

## Storage / Memory / Calldata

| | Storage | Memory | Calldata |
|---|---|---|---|
| Lifetime | permanent (on-chain) | duration of the call | duration of the call |
| Mutability | read/write | read/write | read-only |
| Default for | state variables | function-local variables | external function input (if not `memory`) |
| Relative cost | high, especially writes | low | lowest |

State variables are `storage` by default — every write is permanent and expensive. Function-local variables are `memory` by default — wiped when the call ends, cheap.

The location keyword (`storage` / `memory` / `calldata`) only matters for **reference types** (structs, arrays, mappings, strings) — value types (`uint`, `bool`, `address`) are always copied, so the question doesn't apply to them.

```solidity
function levelUp(uint _id) public {
    Player storage p = idToPlayer[_id]; // alias to the real record
    p.level += 1;                       // mutates storage — costs real gas, persists
}

function previewLevelUp(uint _id) external view returns (uint32) {
    Player memory p = idToPlayer[_id];  // copy, in memory
    p.level += 1;                       // mutates only the copy
    return p.level;                     // original record untouched
}
```

> Same right-hand side, different keyword, completely different semantics and cost. This is a common source of bugs: code that *looks* like a read can silently become a 5,000+ gas write, or vice versa.

---

## Memory Arrays

`memory` arrays must be given a fixed length at creation and **cannot** be resized with `.push()` — only `storage` dynamic arrays support `.push()`.

```solidity
function evensUpTo(uint _max) external pure returns (uint[] memory) {
    uint[] memory evens = new uint[](_max / 2); // fixed-size, in memory
    uint counter = 0;
    for (uint i = 2; i <= _max; i += 2) {
        evens[counter] = i;
        counter++;
    }
    return evens; // discarded after the call returns — never touched storage
}
```

---

## Struct Packing

Storage is allocated in 32-byte slots. Solidity packs **consecutive** fields smaller than 32 bytes into the same slot, but only if their combined size fits — so field order determines slot count.

```solidity
// 2 slots: a and b (uint32 + uint32 = 8 bytes) pack into one slot; c needs its own slot
struct Packed {
    uint32 a;
    uint32 b;
    uint256 c;
}

// 3 slots: c sits between a and b, so neither can pack with the other
struct Unpacked {
    uint32 a;
    uint256 c;
    uint32 b;
}
```

> Packing only helps for fields smaller than 32 bytes — once any field needs a full slot (e.g. `uint256`, a `bytes32`, or any reference type), it always starts a fresh slot and breaks the packing run.

---

## Top-Level State Variable Packing

The same packing rule applies to **plain state variables declared directly in a contract**, not just struct fields. Solidity walks declarations top-to-bottom and packs consecutive small variables into shared slots, exactly as it would inside a struct.

```solidity
contract Registry {
    bool public active = true;                  // slot 0 — next var is full-size, can't share
    uint256 public createdAt = block.timestamp;  // slot 1 (full 32 bytes)
    uint8 private tier = 1;                      // slot 2 ┐
    uint8 private flags = 0;                     // slot 2 │ 1+1+2 = 4 bytes — all three pack together
    uint16 private version = 1;                  // slot 2 ┘
    bytes32[2] private secrets;                  // slots 3, 4
}
```

`active` doesn't pack with `tier`/`flags`/`version` because `createdAt` (a full `uint256`) sits between them in declaration order and forces a new slot boundary on either side of itself.

> This matters beyond gas: anyone can read any storage slot directly via `getStorageAt`, regardless of `public`/`private` (see [Solidity - Forced Ether & On-Chain Privacy Myths](./Solidity%20-%20Forced%20Ether%20%26%20On-Chain%20Privacy%20Myths.md)). Declaration order is the only thing that determines which slot a `private` variable lives in — there's no on-chain name lookup, so this packing rule is exactly what lets you compute the right slot index to read.

---

## Case Study: Rebuild-in-Memory vs Maintain-in-Storage

A recurring design choice: should a contract **store** a derived view of data, or **recompute** it on demand?

**Naive approach** — maintain a per-owner array in storage:

```solidity
mapping (address => uint[]) public ownerToItemIds;
// on mint: ownerToItemIds[owner].push(itemId)
```

`getItemsByOwner` is then a trivial storage read. But now add a `transfer` function. To move `itemId` out of the old owner's array while keeping it gap-free requires either:

1. find the item's index, then shift every subsequent element down one position — a storage **write** per shifted element, or
2. swap-with-last + shrink (cheaper, but reorders the array).

For an owner with 20 items trading away the first, option 1 is 19 storage writes — and the cost is *unpredictable* (depends on how many items the owner has and the index traded), which is its own UX problem: users can't pre-compute a gas limit.

**Better approach** — store nothing extra; rebuild on read:

```solidity
uint[] public allItems;                  // single global array, append-only
mapping (uint => address) public itemOwner;

function getItemsByOwner(address _owner) external view returns (uint[] memory) {
    uint[] memory ids = new uint[](allItems.length); // upper bound, in memory
    uint counter = 0;
    for (uint i = 0; i < allItems.length; i++) {
        if (itemOwner[allItems[i]] == _owner) {
            ids[counter] = allItems[i];
            counter++;
        }
    }
    uint[] memory result = new uint[](counter); // trim to actual size
    for (uint i = 0; i < counter; i++) { result[i] = ids[i]; }
    return result;
}
```

`transfer` now just updates one mapping entry — O(1), fixed cost. `getItemsByOwner` is `external view`: when called as a transaction's entry point, **view functions cost no gas**, regardless of how many iterations the loop runs (see [Solidity - Functions, Visibility & Modifiers](./Solidity%20-%20Functions%2C%20Visibility%20%26%20Modifiers.md) for exactly when `view` is/isn't free).

> Counterintuitive but consistent with the cost table above: an O(n) loop over memory is free externally, while O(n) storage writes are not. Optimize for fewer storage writes, even at the cost of more computation.

---

**See also:** [Solidity - Core Data Types & Structures](./Solidity%20-%20Core%20Data%20Types%20%26%20Structures.md) · [Solidity - Functions, Visibility & Modifiers](./Solidity%20-%20Functions%2C%20Visibility%20%26%20Modifiers.md) · [Solidity - Forced Ether & On-Chain Privacy Myths](./Solidity%20-%20Forced%20Ether%20%26%20On-Chain%20Privacy%20Myths.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
