---
tags: [solidity, smart-contracts, security]
---

Why "predictable randomness" (see [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md)) is an *economic* attack, not just a theoretical flaw — and what real mitigations look like.

---

## The Attack: Simulate Before You Publish

Take a contract that pays out 2x on a "win":

```solidity
function flip() external payable returns (bool won) {
    require(msg.value == 1 ether);
    uint result = uint(keccak256(abi.encodePacked(block.timestamp, msg.sender, nonce++))) % 2;
    if (result == 0) {
        payable(msg.sender).transfer(2 ether);
        return true;
    }
    return false;
}
```

Whoever decides which transactions go into the next block — historically a PoW miner, now a PoS validator or the block builder it works with — can hold this transaction back, compute `result` locally with the exact `block.timestamp` and `nonce` that block will have, and only actually include the transaction in the block if `result == 0`. Losing attempts simply get dropped and retried. From the player's perspective they either win every time they get included, or their transaction never appears at all.

> This is purely an **economics** question: controlling block contents has a cost (running a validator, or having relationships with one). It's only worth doing if the expected payout exceeds that cost. A contract risking a few dollars per flip isn't worth attacking this way; one with `1 ether` bets at scale is.

---

## PoW vs PoS — Same Vulnerability, Different Actor

The flaw isn't tied to Proof-of-Work specifically. What matters is **foreknowledge of block contents before they're final** — whoever has that foreknowledge (PoW miner solving a block, PoS validator proposing one, or an MEV searcher/builder with privileged order-flow access) can apply the same "simulate, then decide whether to include" logic. Ethereum's move to PoS in 2022 changed *who* the privileged actor is, not *whether* one exists.

---

## Real Mitigations

**Chainlink VRF** — a request/fulfill pattern across two transactions. Your contract requests randomness and pays a fee; an oracle network generates a random value off-chain along with a cryptographic proof; a coordinator contract verifies that proof on-chain before calling your contract back with the result. Because the value isn't known to *anyone* — not the requester, not the oracle, not a validator — until after the proof is verified, there's nothing to simulate-and-decide on. Cost: oracle fees, plus the two-transaction latency (the outcome isn't available in the same transaction as the request).

**Commit-Reveal** — two phases. Participants first submit `keccak256(choice, salt)` (hides the actual choice), then later submit `choice, salt` once all commitments are locked in. This doesn't generate randomness by itself, but it stops "see the likely outcome, then decide whether to participate" attacks — useful for sealed bids, votes, or choices that need to be hidden until a deadline.

**`block.prevrandao`** (the post-Merge successor to `block.difficulty`, same opcode/slot) — Ethereum's PoS consensus layer combines contributions from many validators each epoch into a beacon-chain random value, exposed to contracts via this field. Far harder to influence than a single block's hash since no single validator controls the combined result, though the validator revealing last in a round still has a small amount of influence (the "last-revealer" bias). Good enough for lower-stakes randomness; still not recommended alone for high-value outcomes.

| Source | Who provides it | Latency | Typical use |
|---|---|---|---|
| Chainlink VRF | Decentralized oracle network + on-chain proof | 2 transactions (request → fulfill) | Lotteries, NFT mints, high-value gaming |
| Commit-reveal | The participants themselves | 2 transactions, user-driven | Sealed bids, voting, hidden choices |
| `block.prevrandao` | Ethereum validator set (beacon chain) | same transaction, free | Lower-stakes randomness |

---

**See also:** [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) · [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md)
