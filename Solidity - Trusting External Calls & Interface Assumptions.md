---
tags: [solidity, smart-contracts, security]
---

One-line summary: implementing an interface only promises a function's *shape* (name, args, return type) — never its behavior. Calling back into `msg.sender` and trusting it to answer consistently is a real vulnerability class.

---

## Interfaces Are a Promise of Shape, Not Behavior

An `interface` declares a function signature. The compiler enforces that anything cast to that interface *has* a matching function — it enforces nothing about what that function actually does, returns, or whether it returns the same thing twice in a row.

```solidity
interface PriceFeed {
    function latestPrice() external returns (uint256);
}

contract Vault {
    function deposit(uint256 _amount) external {
        PriceFeed feed = PriceFeed(msg.sender); // trusting the caller to BE a PriceFeed
        uint256 price = feed.latestPrice();      // could be anything — no guarantee at all
        // ... logic that assumes `price` is honest
    }
}
```

If `msg.sender` is a wallet or a malicious contract rather than a real, trusted price feed, `latestPrice()` returns whatever its author wants. The interface cast doesn't validate the target's identity or honesty — it only tells the compiler how to format the call.

> The deeper issue: `Vault` has no way to verify `msg.sender` actually *is* a `PriceFeed` it should trust. A cast is not an authentication check. Real systems pin a specific, known-good address (or use a registry/allowlist) rather than trusting whatever calls in.

---

## Non-Idempotent External Calls

A more subtle version of the same problem: a function calls the *same* external method on `msg.sender` more than once, assuming it'll get the same answer both times.

```solidity
interface Membership {
    function isVIP() external returns (bool);
}

contract Lounge {
    bool public admitted;

    function checkIn() public {
        Membership club = Membership(msg.sender);

        if (club.isVIP()) {           // 1st call
            admitted = club.isVIP();  // 2nd call — assumed to match the 1st
        }
    }
}
```

Nothing requires `isVIP()` to return the same value twice. Since you control whatever contract calls `checkIn()`, you also control `isVIP()` — make it return `false` on the first call (to enter the `if`) and `true` on the second (to set `admitted`):

```solidity
contract LoungeAttack {
    bool toggle = true;

    function isVIP() external returns (bool) {
        toggle = !toggle; // flips every call
        return toggle;
    }

    function attack(address _target) external {
        Lounge(_target).checkIn();
    }
}
```

First call: `toggle` flips `true → false`, returns `false` → `!false` is the actual check direction (depends on contract logic), but the principle holds regardless of the exact boolean wiring — different return value per call lets you steer each branch independently.

> Solidity (and most languages) doesn't make any function "pure" by default — a `view`/non-`view` external function can return different values on every call unless you control its code yourself. Never write logic that calls the same external method twice in one function expecting matching results, when that external contract is attacker-controlled.

---

## Audit Checklist

- Flag any external call where the target's address comes from `msg.sender` (or any other untrusted/attacker-influenced address) rather than a hardcoded or registry-verified address.
- Flag any function that calls the **same external method more than once** and uses both results in a security-relevant decision (an `if` gate followed by a state-setting call is the classic shape).
- A cast to an `interface` type is a syntax convenience, never an authentication mechanism — don't read it as one during review.

---

**See also:** [Solidity - Inheritance & Contract Interaction](./Solidity%20-%20Inheritance%20%26%20Contract%20Interaction.md) · [Solidity - Access Control Bypass via Execution Context](./Solidity%20-%20Access%20Control%20Bypass%20via%20Execution%20Context.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
