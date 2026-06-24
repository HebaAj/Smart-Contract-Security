---
tags: [solidity, smart-contracts]
---

One-line summary: function selectors are how the EVM dispatches calls from raw `msg.data`; `abi.encode*` builds that data in Solidity, `web3.eth.abi`/console `sendTransaction` builds and sends it from outside — the syntax and gotchas for actually triggering a contract's logic by hand.

---

## What a function selector is

A signature string is the function name plus parameter types, no spaces, no parameter names — just types: `setX(uint256)`, `claim()`, `transfer(address,uint256)`. Hash it with keccak256, keep the first 4 bytes:

```python
keccak256("setX(uint256)") = 4018d9aad17553f43044109e40e3ad3519b5a0485b2875af7254542c5fec366e
selector                   = 0x4018d9aa   # first 4 bytes only
```

`msg.data` for a call to `setX(42)` is just that selector followed by the ABI-encoded arguments, concatenated:

```
0x4018d9aa  000000000000000000000000000000000000000000000000000000000000002a
└─selector─┘ └────────────── 42, padded to 32 bytes ──────────────────────────┘
```

Compiled contract dispatch is conceptually a big if/else over this 4-byte prefix:

```solidity
bytes4 incoming = bytes4(msg.data[0:4]);
if (incoming == 0x4018d9aa) { /* run setX, decode rest as its argument */ }
else if (incoming == 0x8da5cb5b) { /* run claim */ }
else { /* fallback(), if declared, else revert */ }
```

Overloaded functions (`setX(uint256)` vs `setX(uint256,uint256)`) get different signature strings → different selectors — this is how the EVM tells them apart at the bytecode level.

---

## Building calldata in Solidity

`delegatecall`/`call` take **raw bytes**, not a function-call expression — there's no `contract.func().delegatecall()` chaining. You build the bytes first, then hand them to the low-level call:

```solidity
(bool success, bytes memory returnData) = someAddress.delegatecall(rawBytes);
```

Three ways to build `rawBytes`:

```solidity
// 1. string-based — no compile-time type checking
bytes memory data = abi.encodeWithSignature("setX(uint256)", 42);

// 2. selector computed yourself
bytes4 sel = bytes4(keccak256("setX(uint256)"));
bytes memory data = abi.encodeWithSelector(sel, 42);

// 3. type-checked at compile time (Solidity ≥0.8.11, preferred)
bytes memory data = abi.encodeCall(Lib.setX, (42));
```

All three produce the identical bytes shown above. `encodeCall` needs `Lib`'s interface in scope so the compiler can verify `setX` exists and `42` is the right type — it never actually calls `Lib` normally, it only borrows the signature to build calldata correctly.

To forward calldata you already received, untouched — the pattern behind any unrestricted fallback exploit:

```solidity
fallback() external {
    (bool ok, ) = address(lib).delegatecall(msg.data); // no encoding — already raw bytes
    require(ok);
}
```

---

## Triggering it from outside (console / scripts)

`delegatecall` is an EVM opcode that only exists inside running contract bytecode — an externally-owned account can never issue one directly.

| You want to... | Need a new contract? |
|---|---|
| Trigger a `delegatecall` that already lives inside a deployed contract's code | No — send a transaction with the right calldata |
| Write a `delegatecall` yourself, as part of new logic | Yes — it only appears inside `.sol` code you compile and deploy |

A console can only do one thing to a contract: send it a normal transaction (address, optional ETH, `data` bytes). What the contract's own code does with that — including any internal delegatecall — happens after, on-chain, not from the console.

```js
// JS-side equivalent of abi.encodeWithSignature, hash+slice in one call
let selector = web3.eth.abi.encodeFunctionSignature("pwn()");

await contract.sendTransaction({
  from: player,
  to: contract.address,
  data: selector          // no arguments here — pwn() takes none
});
```

Manual hash-and-slice version of the same thing, for reference:

```js
let fullHash = web3.utils.keccak256("pwn()");   // full 32-byte hash
let selector = fullHash.slice(0, 10);            // "0x" + first 8 hex chars = first 4 bytes
```

> `web3.utils.keccak256(...)` alone is the **full 32-byte hash**, not the selector — sending that whole thing as `data` produces calldata that matches no real function. Always slice to the first 4 bytes (10 characters including `0x`), or use `encodeFunctionSignature` so the slicing is done for you.

---

## Two console gotchas

**`let`/`const` assignments print `undefined`.** That's just how the console echoes statements — the *assignment* evaluates to `undefined`, not the value assigned. The variable itself holds the right value; you just didn't print it.

```js
let selector = web3.eth.abi.encodeFunctionSignature("pwn()");
// undefined          <- expected, ignore

selector
// "0x6a14739f"       <- now it prints
```

**The transaction hash (`tx`) is not the function selector.** They're unrelated values that happen to both show up around the same call.

| | Selector | `tx` hash |
|---|---|---|
| Identifies | which function to run | this specific transaction, uniquely |
| Length | 4 bytes (`0x` + 8 hex chars) | 32 bytes (`0x` + 64 hex chars) |
| Where it appears | inside `data`, as input you constructed | in the return value, after sending |
| Computed from | `keccak256("functionName(types)")` | hash of the whole tx (sender, nonce, data, gas, ...) |

The receipt confirms delivery (`receipt.status`); it doesn't decode or label what you sent. To confirm an exploit worked, check resulting state directly, not the receipt:

```js
await contract.owner() === player   // true means it worked
```

---

## Quick Reference

| Concept | One-liner |
|---|---|
| Function selector | first 4 bytes of `keccak256("name(types)")` — how Solidity dispatches calls |
| `abi.encodeWithSignature` | string-based calldata builder, no compile-time checks |
| `abi.encodeWithSelector` | same, but you supply a pre-computed `bytes4` selector |
| `abi.encodeCall` | type-checked calldata builder (≥0.8.11) — preferred |
| `address.delegatecall(bytes)` | low-level call takes raw bytes, not chained function syntax |
| `web3.eth.abi.encodeFunctionSignature` | JS-side selector builder — hash + slice in one call |
| EOA can't call delegatecall | only running contract bytecode can issue it — console only sends plain transactions |
| `let x = ...` → `undefined` | console echoes the assignment statement, not `x`'s value — print `x` separately |
| `tx` hash ≠ selector | tx hash identifies the transaction; selector identifies the function called |

---

**See also:** [Solidity - delegatecall & Execution Context](./Solidity%20-%20delegatecall%20%26%20Execution%20Context.md) · [Solidity - Functions, Visibility & Modifiers](./Solidity%20-%20Functions%2C%20Visibility%20%26%20Modifiers.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
