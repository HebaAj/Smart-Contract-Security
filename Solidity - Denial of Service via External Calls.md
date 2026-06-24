---
tags: [solidity, smart-contracts, security]
---

One-line summary: gating critical state changes on the success of an external payment lets an attacker permanently lock the function by making the payment fail on purpose.

---

## DoS via Failed External Call

A function that pays out ETH *before* updating its own state assumes that payment will always succeed. An attacker can defeat that assumption by becoming the payment recipient using a contract designed to reject ETH — freezing the function for everyone, forever.

```solidity
contract Throne {
    address public ruler;
    uint256 public fee;

    constructor() payable {
        ruler = msg.sender;
        fee = msg.value;
    }

    receive() external payable {
        require(msg.value >= fee);
        payable(ruler).transfer(msg.value); // (A) pay the old ruler — can be forced to fail
        ruler = msg.sender;                  // (B) only runs if (A) succeeds
        fee = msg.value;                     // (C)
    }
}
```

If `ruler` is a contract with no `receive()`/`fallback()`, step (A) reverts. Solidity rolls back the *entire* transaction on any unhandled failure — so (B) and (C) never execute, and `ruler` can never be displaced.

```solidity
contract Squatter {
    constructor(address payable target) payable {
        (bool sent, ) = target.call{value: msg.value}("");
        require(sent);
    }
    // deliberately no receive()/fallback() — locks the throne permanently
}
```

> `.transfer()` vs `.call()` doesn't change whether this attack works — a recipient with *no* receive/fallback rejects ETH regardless of gas forwarded. `.transfer()`'s fixed 2300-gas stipend only matters if the recipient *has* a receive function but it tries to do gas-heavy logic.

---

## Fix: Pull Over Push

Don't let the contract push payments out as a side effect of someone else's transaction. Let the recipient pull their own funds in a separate transaction instead — a failure on their end then only affects them.

```solidity
contract ThroneFixed {
    address public ruler;
    uint256 public fee;
    mapping(address => uint256) public refunds;

    receive() external payable {
        require(msg.value >= fee);
        if (ruler != address(0)) {
            refunds[ruler] += fee; // credit instead of transferring
        }
        ruler = msg.sender;
        fee = msg.value;
    }

    function withdraw() external {
        uint256 amount = refunds[msg.sender];
        refunds[msg.sender] = 0;
        payable(msg.sender).transfer(amount); // failure here only blocks this caller
    }
}
```

> Audit checklist item: any `require()` or unguarded external call sitting in the middle of a function that needs to keep working for arbitrary future callers is a DoS candidate. Look specifically for ETH transfers to addresses that aren't `msg.sender` and aren't validated as EOAs.

---

**See also:** [Solidity - Forced Ether & On-Chain Privacy Myths](./Solidity%20-%20Forced%20Ether%20%26%20On-Chain%20Privacy%20Myths.md) · [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
