---
tags: [solidity, smart-contracts, security]
---

How contracts receive and hold Ether: `payable`, `msg.value`, the `address`/`address payable` split, and getting funds back out.

---

## `payable` & `msg.value`

A function marked `payable` can receive ETH alongside its call — `msg.value` is the amount sent, in wei:

```solidity
contract CreatorTips is Ownable {
    event TipReceived(address indexed from, uint256 amount);

    function tip() external payable {
        require(msg.value > 0, "send something");
        emit TipReceived(msg.sender, msg.value);
    }
}
```

Without `payable`, the function rejects any call that attaches ETH — the transaction reverts before the function body even runs.

`ether`, `gwei`, and `wei` are unit literals that the compiler converts to plain integers (all amounts are wei under the hood): `1 ether == 1e18`, `1 gwei == 1e9`. So `msg.value == 1 ether` and `msg.value == 1000000000000000000` are identical comparisons.

---

## `address` vs `address payable`

Solidity distinguishes the two types: a plain `address` can be compared, stored, emitted — but **cannot** receive ETH via `.transfer`/`.send`/`.call{value:}`. Only `address payable` can.

```solidity
address payable recipient = payable(owner());  // explicit cast — owner() returns plain `address`
recipient.transfer(1 ether);
```

> CryptoZombies-era code casts via `address(uint160(owner()))` — a round-trip through `uint160` that predates the `payable(...)` conversion function. `payable(addr)` is the direct, current idiom; the `uint160` detour is a historical artifact, not something to reproduce in new code.

---

## Getting ETH Back Out — the Withdraw Pattern

ETH sent to a `payable` function sits in the contract's own balance — `address(this).balance` — with no automatic way out. A withdraw function is the only exit:

```solidity
function withdraw() external onlyOwner {
    payable(owner()).transfer(address(this).balance);
}
```

`onlyOwner` (from [Solidity - Inheritance & Contract Interaction](./Solidity%20-%20Inheritance%20%26%20Contract%20Interaction.md)) is what stops anyone but the contract owner from draining it — without that guard, this function is a "send me all the money" button for the whole world.

---

## `.transfer` vs `.send` vs `.call{value: ...}`

Three ways to actually move the ETH, with materially different failure behavior:

| Method                     | Gas forwarded                             | On failure                                                   |
| -------------------------- | ----------------------------------------- | ------------------------------------------------------------ |
| `.transfer(amount)`        | fixed 2300 stipend                        | reverts the whole transaction                                |
| `.send(amount)`            | fixed 2300 stipend                        | returns `bool` — **silently succeeds if you don't check it** |
| `.call{value: amount}("")` | all remaining gas (or a specified amount) | returns `(bool, bytes)` — must check `bool`                  |

```solidity
(bool ok, ) = payable(owner()).call{value: address(this).balance}("");
require(ok, "withdraw failed");
```

> The 2300-gas stipend on `.transfer`/`.send` was originally a reentrancy defense — too little gas for the recipient's fallback to make another external call. It also breaks legitimate recipients (e.g. multisig wallets) whose fallback needs more than 2300 gas for entirely benign reasons. `.call` is the current recommendation, but forwards enough gas to reopen reentrancy — see below.

---

## Reentrancy

`.call{value:}` forwards enough gas for the recipient to do real work in its `receive()`/`fallback()` — including calling back into the sending contract *before the original call returns*. If state is updated *after* the external call, the recipient sees stale state on that re-entry:

```solidity
contract PerUserVault {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    // VULNERABLE — external call before the state update
    function withdraw() external {
        uint256 amount = balances[msg.sender];
        require(amount > 0);

        (bool ok, ) = msg.sender.call{value: amount}("");
        require(ok);

        balances[msg.sender] = 0;   // too late
    }
}
```

A malicious `msg.sender` is a contract whose `receive()` calls straight back into `withdraw()`:

```solidity
contract Attacker {
    PerUserVault vault;
    constructor(PerUserVault _vault) { vault = _vault; }

    function attack() external payable {
        vault.deposit{value: 1 ether}();
        vault.withdraw();
    }

    receive() external payable {
        if (address(vault).balance >= 1 ether) {
            vault.withdraw();   // re-enter while balances[Attacker] is still 1 ether
        }
    }
}
```

Trace: `withdraw()` reads `balances[Attacker] == 1 ether`, sends 1 ETH via `.call`, which triggers `Attacker.receive()` — still mid-execution of the *first* `withdraw()`, so `balances[Attacker]` hasn't been zeroed yet. `receive()` calls `withdraw()` again; the `require(amount > 0)` check passes again on the same stale balance, sends another 1 ETH, triggers `receive()` again — recursing until the vault is drained or gas runs out.

### Fix: Checks-Effects-Interactions

Update state *before* the external call:

```solidity
function withdraw() external {
    uint256 amount = balances[msg.sender];
    require(amount > 0);          // checks

    balances[msg.sender] = 0;     // effects — BEFORE sending

    (bool ok, ) = msg.sender.call{value: amount}("");
    require(ok);                  // interactions
}
```

Now the re-entrant `withdraw()` sees `balances[Attacker] == 0` and `require(amount > 0)` fails immediately — recursion can't start.

> If `require(ok)` *does* fail here, `balances[msg.sender] = 0` is automatically rolled back too — a revert undoes every state change made during that call, not just the failed step. Solidity never needs manual "undo" logic; either the whole function's effects land, or none of them do.

### `ReentrancyGuard` (defense in depth)

A lock flag that rejects any re-entrant call to a guarded function, regardless of ordering elsewhere in the function:

```solidity
abstract contract ReentrancyGuard {
    bool private locked;

    modifier nonReentrant() {
        require(!locked, "reentrant call");
        locked = true;
        _;
        locked = false;
    }
}
```

```solidity
function withdraw() external nonReentrant {
    // ... as above
}
```

OpenZeppelin's `ReentrancyGuard` is the standard implementation of this. It's a backstop, not a substitute for checks-effects-interactions — use both: ordering prevents the *specific* known reentrancy path, the guard catches anything ordering missed (e.g. a function with multiple external calls or state changes spread across branches).

> Exploit-side gotcha: recursive reentrancy doesn't stop because gas "runs out" in a clean way — the EVM has a hard call-stack depth limit (1024). A low-level `.call()` that exceeds available gas or depth simply returns `(false, ...)` rather than reverting, so a recursive attacker contract without its own stopping condition (e.g. `if (address(target).balance >= amount)`) can silently fail deep calls while shallow ones already succeeded — don't rely on stack-limit exhaustion as your drain condition; check the target's remaining balance explicitly and stop before you'd overshoot it.

---

**See also:** [Solidity - Inheritance & Contract Interaction](./Solidity%20-%20Inheritance%20%26%20Contract%20Interaction.md) · [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
