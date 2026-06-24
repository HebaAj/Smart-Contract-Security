---
tags: [solidity, smart-contracts, security, access-control]
---

One-line summary: three EVM-level signals sometimes used to gate access — `tx.origin`, contract code size, and remaining gas — and exactly how each gets bypassed.

---

## `tx.origin` vs `msg.sender`

`tx.origin` is the externally-owned account (wallet) that started the entire transaction chain, no matter how many contracts it passed through. `msg.sender` is whoever directly called the current function — it changes at every hop.

```solidity
contract Club {
    address public member;

    modifier onlyDirectCaller() {
        require(msg.sender == tx.origin); // intent: "only EOAs, no contracts in between"
        _;
    }

    function join() public onlyDirectCaller {
        member = msg.sender;
    }
}
```

This specific check (`==`) actually *blocks* contract callers, which is itself a common — and discouraged — pattern (it breaks composability with wallets like multisigs and smart-contract wallets). The inverse check, `require(msg.sender != tx.origin)`, is what an attacker uses to *prove* they're calling through a contract:

```solidity
contract ClubAttack {
    function attack(address _target) external {
        Club(_target).join(); // msg.sender (this contract) != tx.origin (your wallet)
    }
}
```

> Real-world direction of this bug: contracts that use `tx.origin` for **authorization** (e.g. `require(tx.origin == owner)`) are phishable — a malicious contract the owner merely *interacts with* can call back into the protected contract, and `tx.origin` still resolves to the owner's wallet. Never use `tx.origin` for authorization; use `msg.sender`.

---

## `extcodesize == 0` Is Not Proof of an EOA

A common (and broken) heuristic for "reject contract callers, only allow wallets": check whether the caller's address has any deployed code.

```solidity
modifier noContracts() {
    uint256 size;
    assembly {
        size := extcodesize(caller()) // low-level equivalent of msg.sender
    }
    require(size == 0, "contracts not allowed");
    _;
}
```

This fails for a specific reason: **during contract construction, the EVM hasn't written the contract's runtime bytecode to its account yet** — that happens only after the constructor finishes and returns the code to store. So `extcodesize` on a contract's own address returns `0` *while its constructor is still running*, even though it's unambiguously a contract.

```solidity
contract Bypass {
    constructor(address _target) {
        // extcodesize(address(this)) == 0 right here — still mid-construction
        ClubWithGuard(_target).join(); // sails through the noContracts() check
    }
}
```

Routing the call through the constructor (rather than a normal function called after deployment) is what makes this work — by the time a post-deployment function could run, code size would already be nonzero.

> `extcodesize` checks are also bypassable in the opposite direction: a wallet can never *fake* having code, but this gate was never reliable for "are you a contract" in the first place. Don't gate logic on code size for security purposes — it's an implementation detail of deployment timing, not an identity proof.

---

## `gasleft()` as an Access Gate

`gasleft()` returns the gas remaining in the current call, at the exact point it's evaluated. Some checks demand it satisfy an arithmetic condition:

```solidity
modifier preciseGas() {
    require(gasleft() % 8191 == 0);
    _;
}
```

There's no way to compute the exact gas remaining by hand — it depends on compiler version, optimizer settings, and the precise bytecode path taken, all the way up to the modifier. The practical approach is to **brute-force the gas supplied to the call** from the attacker's side, trying a range of values until one happens to land on the required remainder:

```solidity
function bruteForce(address _target, bytes8 _key) external {
    for (uint256 i = 0; i < 8191; i++) {
        (bool success, ) = _target.call{gas: i + 150 * 8191}(
            abi.encodeWithSignature("enter(bytes8)", _key)
        );
        if (success) break;
    }
}
```

The base offset (`150 * 8191` here) just needs to supply enough gas for the call to actually reach the modifier — found by experimentation, not derivation. The loop sweeps the last few thousand gas units until the remainder lines up.

> This isn't a "real" vulnerability pattern to look for in audits — gas-based gates like this are intentionally brittle and rarely appear in production contracts. It's included here as the brute-force technique itself is reusable any time a contract's behavior depends on a hard-to-predict, attacker-influenceable EVM value.

---

## Audit Checklist

- `tx.origin` used anywhere in an authorization check (`require(tx.origin == ...)`) — flag immediately, regardless of context.
- Any "block contracts" check (`extcodesize == 0`, or its modern equivalent `address.code.length == 0`) — note that it's bypassable from a constructor and shouldn't be relied on as a security boundary.
- Logic gated on `gasleft()`, `block.gaslimit`, or other execution-environment values an attacker can tune via their own call parameters.

---

**See also:** [Solidity - Trusting External Calls & Interface Assumptions](./Solidity%20-%20Trusting%20External%20Calls%20%26%20Interface%20Assumptions.md) · [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) · [Solidity - delegatecall & Execution Context](./Solidity%20-%20delegatecall%20%26%20Execution%20Context.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
