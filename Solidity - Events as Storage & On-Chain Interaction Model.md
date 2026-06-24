---
tags: [solidity, smart-contracts, gas-optimization, security]
---

Events are cheaper than storage for historical data; `indexed` fields enable efficient filtering. `call` vs `send` maps directly to read-only vs state-changing functions.

---

## Events as Cheap Storage

Writing to state variables is one of the most expensive operations on-chain. Events are far cheaper because they're stored in transaction logs — outside the EVM state trie — and cost a fraction of the gas.

The tradeoff: **events are write-only from the contract's perspective**. A contract cannot read its own past events. They're only accessible off-chain (frontends, indexers, audit tools).

```solidity
// Storing battle outcomes in state: expensive, but contract can query it
mapping(uint => BattleResult) public battleHistory;

// Storing battle outcomes as events: cheap, readable off-chain only
event BattleFought(address indexed attacker, address indexed defender, bool attackerWon);

function attack(uint defenderId) external {
    bool won = _resolveAttack(defenderId);
    // No storage write — just emit; logs cost ~375 gas vs ~20,000 for SSTORE
    emit BattleFought(msg.sender, ownerOf(defenderId), won);
}
```

> Use events for data that the contract itself never needs to compute with — audit trails, historical records, UI feeds. If the contract needs to read it back, it must go in storage.

---

## `indexed` — Filterable Event Parameters

Up to 3 parameters per event can be marked `indexed`. Indexed params are stored as topics in the log, making them efficiently filterable by off-chain consumers without scanning every log.

```solidity
// _from and _to are indexed — filter by either address without scanning all Transfer logs
event Transfer(address indexed _from, address indexed _to, uint256 _tokenId);

// Non-indexed: _tokenId value is in the log data, not searchable as a topic
```

> Auditor relevance: `indexed` fields on events like `Transfer`, `Approval`, and `OwnershipTransferred` are part of the ERC-20/721 spec. Missing `indexed` on required fields is a spec non-compliance finding. Also: events are the primary audit trail for what happened in a contract — if critical state changes (ownership transfers, emergency pauses, large withdrawals) don't emit events, that's a finding.

---

## `call` vs `send` — The Interaction Model

This maps directly to what you already know about `view`/`pure` vs state-changing functions:

| Operation | Used for | Gas cost | Blockchain write |
|---|---|---|---|
| `call` | `view` / `pure` functions | None | No |
| `send` | State-changing functions | Paid by caller | Yes |

`call` executes locally on a node — no transaction, no gas, instant result. `send` creates a transaction that propagates through the network, gets included in a block, and costs gas.

> Auditor relevance: a function that modifies state but is mistakenly marked `view` or `pure` will be callable for free by anyone — the compiler catches most cases, but custom assembly or delegatecall patterns can bypass this. Also: any `send` interaction requires the caller to have ETH for gas, which matters when reasoning about griefing or DoS vectors.

---

**See also:** [Solidity - Functions, Visibility & Modifiers](./Solidity%20-%20Functions%2C%20Visibility%20%26%20Modifiers.md) · [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md) · [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
