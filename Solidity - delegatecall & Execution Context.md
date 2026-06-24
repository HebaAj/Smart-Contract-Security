---
tags: [solidity, smart-contracts, security]
---

One-line summary: `delegatecall` runs another contract's code inside the caller's storage, `msg.sender`, and `msg.value` — the foundation of proxy/library patterns, and an ownership-hijack vector when the forwarded calldata isn't validated (Ethernaut: Delegation).

---

## call vs delegatecall vs staticcall

| | `call` | `delegatecall` | `staticcall` |
|---|---|---|---|
| Code executed | callee's | callee's | callee's |
| Storage written | **callee's** | **caller's** | none (reverts on write) |
| `msg.sender` inside execution | caller's address | unchanged (original sender) | unchanged |
| `msg.value` inside execution | new value passed | unchanged (original value) | unchanged |
| Can send ETH | yes (`{value: x}`) | no — value option unavailable | no |
| Typical use | normal external call | proxy / library pattern | view-only external read |

`call` treats the target as a genuinely separate object: its code runs, its storage gets touched, and it sees you (or your contract) as the caller. `delegatecall` borrows only the target's *code* — every storage operation in that borrowed code gets redirected back into the contract that issued the delegatecall, and `msg.sender`/`msg.value` stay whatever they were on the original call into that contract, unchanged all the way through.

---

## Tracing it through two contracts

```solidity
contract B {
    uint256 public x;        // slot 0
    address public sender;   // slot 1

    function setX(uint256 _x) public {
        x = _x;
        sender = msg.sender;
    }
}

contract A {
    uint256 public x;        // slot 0 — same layout as B
    address public sender;   // slot 1 — same layout as B
    address public bAddr;

    constructor(address _b) { bAddr = _b; }

    function viaCall(uint256 newX) public {
        bAddr.call(abi.encodeWithSignature("setX(uint256)", newX));
    }

    function viaDelegatecall(uint256 newX) public {
        bAddr.delegatecall(abi.encodeWithSignature("setX(uint256)", newX));
    }
}
```

User `0xUser` calls `A.viaCall(42)`:
- execution genuinely jumps into `B`, runs there
- `B.x` → 42, `B.sender` → `address(A)` (A is the immediate caller of B)
- `A.x`, `A.sender` untouched

User `0xUser` calls `A.viaDelegatecall(42)` instead:
- `B`'s bytecode runs, but every storage write it performs lands in **A's storage** — `B`'s own storage is never read or written
- `B`'s code has no concept of "A" or "B" by name — `x = _x` compiles to "write to slot 0," and slot 0 belongs to whichever contract's storage context the call is executing in
- `A.x` → 42 (A's slot 0, even though `viaDelegatecall` never wrote `x` itself)
- `A.sender` → `0xUser` — the *original* caller, not `A`, because `msg.sender` never changes across a delegatecall chain
- `B.x`, `B.sender` untouched

> Storage layout must match between caller and callee for delegatecall to be safe — slots are addressed by index, not name. If `B` declared `sender` before `x` (slots swapped relative to `A`), `B.setX`'s write to "slot 1" would silently land in `A.sender` instead of `A.x` — no revert, no warning, just corrupted state of the wrong type.

---

## Why this enables on-chain libraries

A library contract holding only logic (no meaningful state of its own) can be deployed once and delegatecalled into by many proxy contracts, each keeping its own independent storage. This is the basis of upgradeable proxy patterns: storage lives permanently in the proxy, the logic contract's address can be swapped out, and every call is forwarded via delegatecall so execution always behaves as if it were the proxy's own code.

```solidity
contract MathLib {
    // no meaningful state — only ever delegatecall'd into
    function double(uint256 x) external pure returns (uint256) {
        return x * 2;
    }
}

contract Proxy {
    uint256 public result;
    address public lib;

    constructor(address _lib) { lib = _lib; }

    function compute(uint256 x) external {
        (bool ok, bytes memory data) = lib.delegatecall(
            abi.encodeWithSignature("double(uint256)", x)
        );
        require(ok);
        result = abi.decode(data, (uint256)); // Proxy's own storage, not MathLib's
    }
}
```

`MathLib` doesn't need to know how many proxies point at it, or that it's being delegatecalled rather than called normally — it's just bytecode sitting at an address, waiting to be borrowed.

---

## Fallback dispatch: forwarding calls Router never declared

Solidity dispatches incoming calls by matching the first 4 bytes of `msg.data` (the function selector) against each declared function. If nothing matches, execution falls to `fallback()` if one exists — with the *entire* `msg.data` still intact.

```solidity
contract Lib {
    address public owner;   // slot 0

    function claim() public { owner = msg.sender; }
}

contract Router {
    address public owner;   // slot 0 — matches Lib's slot 0
    Lib lib;

    constructor(address _lib) { lib = Lib(_lib); owner = msg.sender; }

    fallback() external {
        (bool ok, ) = address(lib).delegatecall(msg.data);
        require(ok);
    }
}
```

`Router` has no `claim()` function. A call whose selector matches `claim()` finds no match on `Router` → falls to `fallback()` → `fallback()` forwards that exact `msg.data` into `Lib` via delegatecall.

Tracing the exploit: attacker `0xEvil` sends a transaction to `Router` with `data` set to `claim()`'s selector, no arguments.
1. `Router` checks its own functions — no match for `claim()`'s selector.
2. Falls to `fallback()`.
3. `fallback()` delegatecalls `Lib` with the untouched `msg.data`.
4. `Lib`'s code for `claim()` runs, but in `Router`'s storage context — `msg.sender` inside that execution is still `0xEvil` (unchanged since the original call into `Router`).
5. `claim()`'s body writes `msg.sender` to slot 0. Slot 0 in `Router` is `Router.owner`.
6. `Router.owner` becomes `0xEvil`. `Lib.owner` (Lib's own slot 0) is never touched — `Lib`'s storage is never read or written in this whole chain.

The root cause isn't delegatecall itself — it's that `Router.fallback()` forwards *any* attacker-chosen calldata with zero check on which selector is allowed through.

> `delegatecall` is an EVM opcode that only exists inside running contract bytecode — an externally-owned account (a wallet, a console) can never issue one directly. The attacker's transaction above is a plain `call` into `Router`; the delegatecall into `Lib` happens *inside* `Router`'s own code, triggered indirectly. See [Solidity - ABI Encoding, Selectors & Console Interaction](./Solidity%20-%20ABI%20Encoding%2C%20Selectors%20%26%20Console%20Interaction.md) for how to actually send a transaction like this from a console.

---

## Quick Reference

| Concept | One-liner |
|---|---|
| `delegatecall` context | runs callee's *code*, keeps caller's storage / `msg.sender` / `msg.value` |
| Storage slot alignment | callee writes by slot index — caller and callee layouts must match or storage gets clobbered |
| Library / proxy pattern | logic contract holds no real state, proxy's storage is permanent, calls forwarded via delegatecall |
| `fallback()` trigger | runs when no function selector in `msg.data` matches any declared function |
| Unrestricted delegatecall-in-fallback | forwards arbitrary attacker-chosen calldata into caller's own storage — root cause of the Delegation exploit |
| delegatecall is contract-only | EOAs/consoles can never issue delegatecall directly — only running contract bytecode can |

---

**See also:** [Solidity - ABI Encoding, Selectors & Console Interaction](./Solidity%20-%20ABI%20Encoding%2C%20Selectors%20%26%20Console%20Interaction.md) · [Solidity - Functions, Visibility & Modifiers](./Solidity%20-%20Functions%2C%20Visibility%20%26%20Modifiers.md) · [Solidity - Inheritance & Contract Interaction](./Solidity%20-%20Inheritance%20%26%20Contract%20Interaction.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
