---
tags: [solidity, smart-contracts, security, gas-optimization]
---

Integer overflow/underflow in Solidity — how silent wrapping creates exploitable vulnerabilities, and how SafeMath and Solidity 0.8+ protect against it.

---

## The Problem: Silent Wrapping

Solidity integers are fixed-width. When arithmetic exceeds the type's range, the value wraps silently — no error, no revert.

```solidity
uint8 x = 255;
x += 1; // wraps to 0 — no error thrown

uint8 y = 0;
y -= 1; // wraps to 255 — uint can't go negative, so it wraps to max
```

This is inherited from low-level integer behavior. Before Solidity 0.8, *every* arithmetic operation had this risk.

> Classic exploit: a contract tracks token balances as `uint`. If an attacker can trigger `balance -= amount` where `amount > balance`, the result wraps to a huge number, giving them effectively unlimited tokens.

---

## SafeMath — The Pre-0.8 Fix

OpenZeppelin's `SafeMath` is a library that wraps arithmetic with overflow checks using `assert`. It uses the `using X for Y` pattern to attach methods directly to a type.

```solidity
using SafeMath for uint256;

uint256 supply = 1000;
// These revert on overflow/underflow instead of wrapping silently
uint256 newSupply = supply.add(500);   // checks: result >= a
uint256 reduced   = supply.sub(200);   // checks: a >= b before subtracting
uint256 doubled   = supply.mul(2);     // checks: a == 0 || result / a == b
uint256 half      = supply.div(2);     // checks: b != 0
```

How `add` actually works internally:

```solidity
function add(uint256 a, uint256 b) internal pure returns (uint256) {
    uint256 c = a + b;
    assert(c >= a); // if overflow occurred, c < a — this catches it
    return c;
}
```

`assert` vs `require` matters here: `assert` consumes all remaining gas on failure (signals "this should never happen — something is fundamentally broken"), whereas `require` refunds unused gas (signals "expected failure condition"). SafeMath uses `assert` because an overflow is a logic error in the contract, not a user input error.

---

## Per-Type SafeMath

`SafeMath` operates on `uint256`. If your contract uses smaller types (`uint32`, `uint16`), you need separate library instances — otherwise the value gets promoted to `uint256` for the operation and the smaller type's bounds are never checked.

```solidity
// Won't catch uint32 overflow — arithmetic happens in uint256 space
using SafeMath for uint32; // ← wrong: SafeMath is defined for uint256

// Correct approach pre-0.8: define SafeMath32, SafeMath16 with matching types
library SafeMath32 {
    function add(uint32 a, uint32 b) internal pure returns (uint32) {
        uint32 c = a + b;
        assert(c >= a);
        return c;
    }
    // sub, mul, div similarly
}
```

> This is a common oversight: a contract uses `SafeMath` for its main balances (`uint256`) but has a `uint32` counter elsewhere that overflows silently.

---

## Solidity 0.8+ — Built-in Checks

From 0.8 onward, overflow and underflow revert automatically on every arithmetic operation. SafeMath is no longer needed for default arithmetic.

```solidity
// In Solidity ^0.8.0 — this reverts automatically, no SafeMath required
uint8 x = 255;
x += 1; // reverts with Panic(0x11)
```

If you explicitly need wrapping behavior (e.g. cryptographic operations, ring counters), use the `unchecked` block — but only when you've verified it's safe:

```solidity
unchecked {
    x += 1; // wraps without revert — opt-in, intentional
}
```

> Auditor flag: `unchecked` blocks in modern Solidity are high-attention areas. Any arithmetic inside one must be manually verified safe. An `unchecked` subtraction where the result could underflow is a critical finding.

> Debugging gotcha: a `SafeMath`-guarded `add()` reverting doesn't always mean *this* call is wrong — if a prior transaction already pushed a balance close to `type(uint256).max` (e.g. via an earlier reentrancy attack that underflowed `balances[x] -= amount` before being fixed), any later legitimate addition to that same `balances[x]` will overflow and revert too. Check the existing stored value before assuming the new call's logic is the problem — especially when re-testing an exploit against the same address after a partial/failed first attempt.

---

## Quick Reference

| Context | Behavior |
|---|---|
| Solidity < 0.8, no SafeMath | Silent wrap — exploitable |
| Solidity < 0.8, with SafeMath | Reverts via `assert` on overflow |
| Solidity ≥ 0.8, default | Reverts automatically (Panic) |
| Solidity ≥ 0.8, `unchecked {}` | Silent wrap — intentional opt-in |

---

**See also:** [Solidity - Core Data Types & Structures](./Solidity%20-%20Core%20Data%20Types%20%26%20Structures.md) · [Solidity - Functions, Visibility & Modifiers](./Solidity%20-%20Functions%2C%20Visibility%20%26%20Modifiers.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
