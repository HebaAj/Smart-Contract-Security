---
tags: [solidity, smart-contracts]
---

One-line summary: how Solidity turns typed values into bytes for hashing/comparison, the explicit conversions between types, and how truncating casts can be reverse-engineered into access-control puzzles. Living reference — grows as new patterns appear.

---

## String/Bytes Equality via `keccak256`

No native `==` for `string`/`bytes`/arrays. Standard pattern: hash both sides and compare hashes.

```solidity
function isAdmin(string memory _name) private pure returns (bool) {
    return keccak256(abi.encodePacked(_name)) == keccak256(abi.encodePacked("admin"));
}
```

`keccak256` takes a single `bytes` argument — `abi.encodePacked(...)` is how arbitrary values get turned into that `bytes` blob.

---

## `abi.encodePacked` vs `abi.encode`

`encodePacked` concatenates the byte representations of its arguments with **no padding and no length prefixes**. That's compact, but different inputs can produce identical output:

```solidity
abi.encodePacked("a", "bc") == abi.encodePacked("ab", "c") // true — both = 0x616263
```

So `keccak256(abi.encodePacked("a","bc"))` and `keccak256(abi.encodePacked("ab","c"))` collide.

`abi.encode(...)` pads each argument to 32 bytes and includes lengths — collision-free, at the cost of larger output.

> If a contract hashes multiple **variable-length** values together (e.g. for a signature, a commitment, a unique ID), `encodePacked` over those values is a real vulnerability — use `encode` instead. `encodePacked` is fine for fixed-length values or a single argument.

---

## Explicit Type Conversion

Solidity won't implicitly narrow types — casts must be explicit, and narrowing **truncates** rather than erroring.

```solidity
uint256 ts = block.timestamp;
uint32 ts32 = uint32(ts); // truncates to the lower 32 bits — silent data loss if ts > 2^32-1
```

> Widening (`uint8` → `uint256`) is always safe. Narrowing (`uint256` → `uint32`) silently drops high-order bits — only do this once you've reasoned about the value's range (e.g. timestamps until 2106).

---

## Truncation Direction: Integers vs Bytes

The two truncating-cast families drop bits from **opposite ends** — conflating them produces the wrong slice of data.

- **Narrowing a `uintN`** keeps the **low-order (rightmost)** bits, drops the high-order ones.
- **Narrowing a `bytesN`** keeps the **high-order (leftmost)** bytes, drops the trailing ones.

```solidity
uint256 fullId   = 0x00000000000000000000000000000000000000000000000000123456789ABC;
uint32  lowSlice = uint32(fullId);   // last 4 bytes → keeps the RIGHT end

bytes32 fullKey  = 0x123456789ABCDEF0000000000000000000000000000000000000000000000;
bytes16 hiSlice  = bytes16(fullKey); // keeps the LEFT 16 bytes, drops the rest
```

> Easy bug: assuming a `bytesN` cast behaves like a `uintN` cast (or vice versa) and reading the wrong half of a value entirely. Always check which family you're casting before reasoning about "which part survives."

---

## Multi-Stage Truncation as a Constraint Puzzle

Some access-control checks chain several narrowing casts of the same value and compare the results against each other. Each cast exposes a different "zone" of bits, and the constraints between them can pin down exactly which bits are free to choose.

```solidity
// Hypothetical gate: only passes for a _key whose bit zones satisfy all three checks
function passGate(uint64 _key) external pure returns (bool) {
    require(uint32(_key) == uint16(_key));   // zone 16–31 of _key must be all zero
    require(uint32(_key) != _key);           // some bit in zone 32–63 must be nonzero
    return true;
}
```

Splitting `_key` into four 16-bit zones (bits 0–15, 16–31, 32–47, 48–63) makes the constraints readable: the first `require` forces zone 16–31 to zero (since `uint32` must equal `uint16` of the same value); the second forces *something* above bit 31 to be set. Zones 0–15 and 16–31 are fully constrained; zones 32–63 are completely free as long as one bit there is `1` — pick the simplest one (e.g. bit 32) to satisfy it.

> When solving these, draw the bit zones out explicitly. The casts tell you exactly which zones each `require` can "see" — bits outside every checked zone are free.

---

## XOR as a Reversible Constraint

XOR is its own inverse: if `a ^ b == c`, then `b == a ^ c`. This turns "find a value that XORs with something fixed to produce a target" into a direct computation rather than a search.

```solidity
// Hypothetical gate: caller must supply a key that XORs with their own hash to all-1s
function passGate(bytes8 _key) external view returns (bool) {
    bytes8 hashed = bytes8(keccak256(abi.encodePacked(msg.sender)));
    return uint64(hashed) ^ uint64(_key) == type(uint64).max;
}
```

`type(uint64).max` is every bit set to `1`. XOR-ing any value against all-1s flips every bit (bitwise NOT). So the required key is simply `~hashed`:

```solidity
bytes8 key = bytes8(uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ type(uint64).max);
```

> Recognize `^ type(uintN).max` as "compute the complement, then solve the XOR algebraically" — no brute force needed, unlike `gasleft() % N` style gates (see [Solidity - Access Control Bypass via Execution Context](./Solidity%20-%20Access%20Control%20Bypass%20via%20Execution%20Context.md)).

---

## `keccak256` as "Randomness" — and Why It Isn't

`keccak256` is a real hash function (a SHA-3 variant), often reached for as a source of pseudo-randomness:

```solidity
uint randomish = uint(keccak256(abi.encodePacked(block.timestamp, msg.sender))) % 100;
```

This is **predictable and manipulable**: every input here — `block.timestamp`, `msg.sender`, and similarly `block.difficulty` / `blockhash` — is either known in advance or chosen by whoever produces the block. Anyone, including the miner/validator, can compute the result before deciding whether to include the transaction, or can call from a contract that checks the result first and reverts if unfavorable.

> Flag any on-chain "randomness" derived from block/transaction data as a likely vulnerability. Real randomness needs an external oracle (e.g. Chainlink VRF).

---

**See also:** [Solidity - Core Data Types & Structures](./Solidity%20-%20Core%20Data%20Types%20%26%20Structures.md) · [Solidity - Access Control Bypass via Execution Context](./Solidity%20-%20Access%20Control%20Bypass%20via%20Execution%20Context.md) · [Solidity - Forced Ether & On-Chain Privacy Myths](./Solidity%20-%20Forced%20Ether%20%26%20On-Chain%20Privacy%20Myths.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
