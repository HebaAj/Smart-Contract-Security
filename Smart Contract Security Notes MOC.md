---
tags: [solidity, smart-contracts, moc]
---

Map of content for smart contract security and Solidity notes.
---

## Notes

| Note | Covers |
|---|---|
| [Solidity - Core Data Types & Structures](./Solidity%20-%20Core%20Data%20Types%20%26%20Structures.md) | `uint` family, structs, arrays, mappings, strings, time |
| [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) | `keccak256` equality pattern, `abi.encodePacked` vs `encode`, typecasting, truncation direction (int vs bytes), multi-stage bit-zone puzzles, XOR self-inverse, on-chain randomness pitfalls |
| [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md) | storage/memory/calldata, gas cost model, struct packing, top-level state variable packing, memory arrays, rebuild-vs-maintain pattern |
| [Solidity - Functions, Visibility & Modifiers](./Solidity%20-%20Functions%2C%20Visibility%20%26%20Modifiers.md) | public/private/internal/external, view/pure, constructors, modifiers, events |
| [Solidity - Inheritance & Contract Interaction](./Solidity%20-%20Inheritance%20%26%20Contract%20Interaction.md) | inheritance, `Ownable` pattern, `msg.sender`/`require`, addresses, interfaces |
| [Solidity - Trusting External Calls & Interface Assumptions](./Solidity%20-%20Trusting%20External%20Calls%20%26%20Interface%20Assumptions.md) | interfaces as shape-only promises, untrusted `msg.sender` casts, non-idempotent external calls, Ethernaut Elevator exploit |
| [Solidity - delegatecall & Execution Context](./Solidity%20-%20delegatecall%20%26%20Execution%20Context.md) | `call` vs `delegatecall` vs `staticcall`, storage slot alignment, on-chain libraries/proxies, fallback dispatch, Ethernaut Delegation exploit trace |
| [Solidity - Access Control Bypass via Execution Context](./Solidity%20-%20Access%20Control%20Bypass%20via%20Execution%20Context.md) | `tx.origin` vs `msg.sender`, `extcodesize == 0` during constructor execution, `gasleft()` brute-forcing, Ethernaut Gatekeeper One/Two exploits |
| [Solidity - ABI Encoding, Selectors & Console Interaction](./Solidity%20-%20ABI%20Encoding%2C%20Selectors%20%26%20Console%20Interaction.md) | function selectors, `abi.encodeWithSignature`/`encodeWithSelector`/`encodeCall`, EOA vs contract delegatecall limits, web3 console syntax, tx-hash-vs-selector gotcha |
| [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) | `payable`, `msg.value`, `address payable`, withdraw pattern, reentrancy + ReentrancyGuard |
| [Solidity - Randomness Attacks & Mitigations](./Solidity%20-%20Randomness%20Attacks%20%26%20Mitigations.md) | `keccak256(block-data)` predictability, simulate-before-publish, Chainlink VRF, commit-reveal, `block.prevrandao` |
| [Solidity - Integer Overflow & SafeMath](./Solidity%20-%20Integer%20Overflow%20%26%20SafeMath.md) | silent wrapping pre-0.8, SafeMath library, checked arithmetic in ≥0.8, `unchecked` blocks |
| [Solidity - Token Standards (ERC-20 & ERC-721)](./Solidity%20-%20Token%20Standards%20%28ERC-20%20%26%20ERC-721%29.md) | fungible vs non-fungible, standard interfaces, two-step approval flow, composability, zero-address burn guard, partial transfer/transferFrom override gap |
| [Solidity - Events as Storage & On-Chain Interaction Model](./Solidity%20-%20Events%20as%20Storage%20%26%20On-Chain%20Interaction%20Model.md) | events vs storage gas tradeoff, `indexed` fields, `call` vs `send` mapped to view/pure vs state-changing |
| [Solidity - Forced Ether & On-Chain Privacy Myths](./Solidity%20-%20Forced%20Ether%20%26%20On-Chain%20Privacy%20Myths.md) | `selfdestruct` forced ETH transfer, balance-invariant breakage, `private` storage transparency, packed-slot extraction worked example, commit-reveal/ZK alternatives |
| [Solidity - Denial of Service via External Calls](./Solidity%20-%20Denial%20of%20Service%20via%20External%20Calls.md) | push-payment DoS pattern, `.transfer()` vs `.call()` gas stipend, pull-over-push fix |

---

## Concept Quick Reference

| Concept | Note |
|---|---|
| `uint` family & sub-types (no gas benefit alone) | [Solidity - Core Data Types & Structures](./Solidity%20-%20Core%20Data%20Types%20%26%20Structures.md) |
| Structs as field bundles | [Solidity - Core Data Types & Structures](./Solidity%20-%20Core%20Data%20Types%20%26%20Structures.md) |
| Fixed vs dynamic arrays, public auto-getters | [Solidity - Core Data Types & Structures](./Solidity%20-%20Core%20Data%20Types%20%26%20Structures.md) |
| Mappings (no iteration, zero-value default) | [Solidity - Core Data Types & Structures](./Solidity%20-%20Core%20Data%20Types%20%26%20Structures.md) |
| Time: `block.timestamp`, time-unit literals | [Solidity - Core Data Types & Structures](./Solidity%20-%20Core%20Data%20Types%20%26%20Structures.md) |
| `keccak256` for equality comparison | [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) |
| `abi.encodePacked` vs `abi.encode` (collision risk) | [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) |
| Explicit typecasting (`uint256(x)`) | [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) |
| Truncation direction: `uintN` keeps low bits, `bytesN` keeps high bytes | [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) |
| Multi-stage truncation as a bit-zone constraint puzzle | [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) |
| XOR self-inverse (`a^b==c` ⟹ `b==a^c`) to solve bitmask gates | [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) |
| On-chain randomness pitfalls (`keccak256(block.*)`) | [Solidity - Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) |
| Storage vs memory vs calldata — cost model | [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md) |
| Struct field packing (order matters for slots) | [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md) |
| Top-level state variable packing (same rule outside structs) | [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md) |
| Memory arrays (fixed-size only, no `push`) | [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md) |
| Rebuild-vs-maintain pattern for gas | [Solidity - Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md) |
| public/private/internal/external visibility | [Solidity - Functions, Visibility & Modifiers](./Solidity%20-%20Functions%2C%20Visibility%20%26%20Modifiers.md) |
| view / pure / payable state mutability | [Solidity - Functions, Visibility & Modifiers](./Solidity%20-%20Functions%2C%20Visibility%20%26%20Modifiers.md) |
| Custom modifiers (`modifier`, `_`) | [Solidity - Functions, Visibility & Modifiers](./Solidity%20-%20Functions%2C%20Visibility%20%26%20Modifiers.md) |
| Events & indexed parameters | [Solidity - Functions, Visibility & Modifiers](./Solidity%20-%20Functions%2C%20Visibility%20%26%20Modifiers.md) |
| Inheritance & `super` | [Solidity - Inheritance & Contract Interaction](./Solidity%20-%20Inheritance%20%26%20Contract%20Interaction.md) |
| `Ownable` pattern (`onlyOwner` modifier) | [Solidity - Inheritance & Contract Interaction](./Solidity%20-%20Inheritance%20%26%20Contract%20Interaction.md) |
| `msg.sender` + `require` for access control | [Solidity - Inheritance & Contract Interaction](./Solidity%20-%20Inheritance%20%26%20Contract%20Interaction.md) |
| Interfaces for cross-contract calls | [Solidity - Inheritance & Contract Interaction](./Solidity%20-%20Inheritance%20%26%20Contract%20Interaction.md) |
| Interfaces promise shape, not behavior | [Solidity - Trusting External Calls & Interface Assumptions](./Solidity%20-%20Trusting%20External%20Calls%20%26%20Interface%20Assumptions.md) |
| Untrusted `msg.sender` cast to an interface type | [Solidity - Trusting External Calls & Interface Assumptions](./Solidity%20-%20Trusting%20External%20Calls%20%26%20Interface%20Assumptions.md) |
| Non-idempotent external calls (same call, different answers) | [Solidity - Trusting External Calls & Interface Assumptions](./Solidity%20-%20Trusting%20External%20Calls%20%26%20Interface%20Assumptions.md) |
| Ethernaut Elevator exploit (toggled `isLastFloor` return) | [Solidity - Trusting External Calls & Interface Assumptions](./Solidity%20-%20Trusting%20External%20Calls%20%26%20Interface%20Assumptions.md) |
| `delegatecall` context (caller's storage/`msg.sender`/`msg.value`, callee's code) | [Solidity - delegatecall & Execution Context](./Solidity%20-%20delegatecall%20%26%20Execution%20Context.md) |
| `call` vs `delegatecall` vs `staticcall` | [Solidity - delegatecall & Execution Context](./Solidity%20-%20delegatecall%20%26%20Execution%20Context.md) |
| Storage slot alignment requirement for delegatecall | [Solidity - delegatecall & Execution Context](./Solidity%20-%20delegatecall%20%26%20Execution%20Context.md) |
| On-chain library / proxy pattern via delegatecall | [Solidity - delegatecall & Execution Context](./Solidity%20-%20delegatecall%20%26%20Execution%20Context.md) |
| `fallback()` trigger on unmatched selector | [Solidity - delegatecall & Execution Context](./Solidity%20-%20delegatecall%20%26%20Execution%20Context.md) |
| Ethernaut Delegation exploit (unrestricted fallback delegatecall → storage clobber) | [Solidity - delegatecall & Execution Context](./Solidity%20-%20delegatecall%20%26%20Execution%20Context.md) |
| `tx.origin` vs `msg.sender` — phishing & authorization risk | [Solidity - Access Control Bypass via Execution Context](./Solidity%20-%20Access%20Control%20Bypass%20via%20Execution%20Context.md) |
| `extcodesize == 0` bypassed during constructor execution | [Solidity - Access Control Bypass via Execution Context](./Solidity%20-%20Access%20Control%20Bypass%20via%20Execution%20Context.md) |
| `gasleft() % N` gate — brute-forcing the supplied gas | [Solidity - Access Control Bypass via Execution Context](./Solidity%20-%20Access%20Control%20Bypass%20via%20Execution%20Context.md) |
| Ethernaut GatekeeperOne/GatekeeperTwo exploits | [Solidity - Access Control Bypass via Execution Context](./Solidity%20-%20Access%20Control%20Bypass%20via%20Execution%20Context.md) |
| Function selector (`keccak256("name(types)")[:4]`) | [Solidity - ABI Encoding, Selectors & Console Interaction](./Solidity%20-%20ABI%20Encoding%2C%20Selectors%20%26%20Console%20Interaction.md) |
| `abi.encodeWithSignature` / `encodeWithSelector` / `encodeCall` | [Solidity - ABI Encoding, Selectors & Console Interaction](./Solidity%20-%20ABI%20Encoding%2C%20Selectors%20%26%20Console%20Interaction.md) |
| `address.delegatecall(bytes)` / `.call(bytes)` syntax — raw bytes, not chained calls | [Solidity - ABI Encoding, Selectors & Console Interaction](./Solidity%20-%20ABI%20Encoding%2C%20Selectors%20%26%20Console%20Interaction.md) |
| EOA/console can't issue delegatecall directly — only running contract bytecode can | [Solidity - ABI Encoding, Selectors & Console Interaction](./Solidity%20-%20ABI%20Encoding%2C%20Selectors%20%26%20Console%20Interaction.md) |
| `web3.eth.abi.encodeFunctionSignature` + `sendTransaction({data})` console workflow | [Solidity - ABI Encoding, Selectors & Console Interaction](./Solidity%20-%20ABI%20Encoding%2C%20Selectors%20%26%20Console%20Interaction.md) |
| Console `let x = ...` prints `undefined` (echoes statement, not value) | [Solidity - ABI Encoding, Selectors & Console Interaction](./Solidity%20-%20ABI%20Encoding%2C%20Selectors%20%26%20Console%20Interaction.md) |
| Tx hash vs function selector — unrelated values, different lengths/purposes | [Solidity - ABI Encoding, Selectors & Console Interaction](./Solidity%20-%20ABI%20Encoding%2C%20Selectors%20%26%20Console%20Interaction.md) |
| `payable` functions & `msg.value` | [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) |
| Receiving ETH | [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) |
| `ether`/`gwei`/`wei` unit literals | [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) |
| `address` vs `address payable` (+ outdated `uint160` cast) | [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) |
| `.transfer`/`.send`/`.call{value:}` — gas stipend & reentrancy tradeoff | [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) |
| Withdraw pattern (`address(this).balance`, `onlyOwner`) | [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) |
| Reentrancy (external call before state update) | [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) |
| Checks-effects-interactions ordering | [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) |
| Revert = atomic rollback of all state changes | [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) |
| `ReentrancyGuard` / `nonReentrant` | [Solidity - Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) |
| Simulate-before-publish randomness exploit | [Solidity - Randomness Attacks & Mitigations](./Solidity%20-%20Randomness%20Attacks%20%26%20Mitigations.md) |
| Block-producer foreknowledge (PoW miners & PoS validators alike) | [Solidity - Randomness Attacks & Mitigations](./Solidity%20-%20Randomness%20Attacks%20%26%20Mitigations.md) |
| Chainlink VRF (request/fulfill + on-chain proof) | [Solidity - Randomness Attacks & Mitigations](./Solidity%20-%20Randomness%20Attacks%20%26%20Mitigations.md) |
| Commit-reveal scheme | [Solidity - Randomness Attacks & Mitigations](./Solidity%20-%20Randomness%20Attacks%20%26%20Mitigations.md) |
| `block.prevrandao` (RANDAO) | [Solidity - Randomness Attacks & Mitigations](./Solidity%20-%20Randomness%20Attacks%20%26%20Mitigations.md) |
| Integer overflow / underflow — silent wrapping pre-0.8 | [Solidity - Integer Overflow & SafeMath](./Solidity%20-%20Integer%20Overflow%20%26%20SafeMath.md) |
| SafeMath library (`using X for T`, `assert` vs `require`) | [Solidity - Integer Overflow & SafeMath](./Solidity%20-%20Integer%20Overflow%20%26%20SafeMath.md) |
| Checked arithmetic default in Solidity ≥0.8 | [Solidity - Integer Overflow & SafeMath](./Solidity%20-%20Integer%20Overflow%20%26%20SafeMath.md) |
| `unchecked` block — where overflow bugs now hide | [Solidity - Integer Overflow & SafeMath](./Solidity%20-%20Integer%20Overflow%20%26%20SafeMath.md) |
| Per-type SafeMath (`SafeMath16`, `SafeMath32`) | [Solidity - Integer Overflow & SafeMath](./Solidity%20-%20Integer%20Overflow%20%26%20SafeMath.md) |
| ERC20 — fungible token standard & interface | [Solidity - Token Standards (ERC-20 & ERC-721)](./Solidity%20-%20Token%20Standards%20%28ERC-20%20%26%20ERC-721%29.md) |
| ERC721 — non-fungible token standard & interface | [Solidity - Token Standards (ERC-20 & ERC-721)](./Solidity%20-%20Token%20Standards%20%28ERC-20%20%26%20ERC-721%29.md) |
| Two-step approval flow (`approve` + `transferFrom`) | [Solidity - Token Standards (ERC-20 & ERC-721)](./Solidity%20-%20Token%20Standards%20%28ERC-20%20%26%20ERC-721%29.md) |
| Standard composability — why conforming beats rolling your own | [Solidity - Token Standards (ERC-20 & ERC-721)](./Solidity%20-%20Token%20Standards%20%28ERC-20%20%26%20ERC-721%29.md) |
| Zero-address burn guard (`require(_to != address(0))`) | [Solidity - Token Standards (ERC-20 & ERC-721)](./Solidity%20-%20Token%20Standards%20%28ERC-20%20%26%20ERC-721%29.md) |
| Partial override gap — restricting `transfer` without restricting `transferFrom` | [Solidity - Token Standards (ERC-20 & ERC-721)](./Solidity%20-%20Token%20Standards%20%28ERC-20%20%26%20ERC-721%29.md) |
| Ethernaut NaughtCoin exploit (`approve` + `transferFrom` bypasses timelocked `transfer`) | [Solidity - Token Standards (ERC-20 & ERC-721)](./Solidity%20-%20Token%20Standards%20%28ERC-20%20%26%20ERC-721%29.md) |
| Events vs storage — gas tradeoff & write-only constraint | [Solidity - Events as Storage & On-Chain Interaction Model](./Solidity%20-%20Events%20as%20Storage%20%26%20On-Chain%20Interaction%20Model.md) |
| `indexed` event params — filterable topics vs log data | [Solidity - Events as Storage & On-Chain Interaction Model](./Solidity%20-%20Events%20as%20Storage%20%26%20On-Chain%20Interaction%20Model.md) |
| Missing events on critical state changes (audit finding) | [Solidity - Events as Storage & On-Chain Interaction Model](./Solidity%20-%20Events%20as%20Storage%20%26%20On-Chain%20Interaction%20Model.md) |
| `call` (read-only, no gas) vs `send` (state-changing, costs gas) | [Solidity - Events as Storage & On-Chain Interaction Model](./Solidity%20-%20Events%20as%20Storage%20%26%20On-Chain%20Interaction%20Model.md) |
| `selfdestruct` forced ETH transfer (bypasses receive/fallback/payable) | [Solidity - Forced Ether & On-Chain Privacy Myths](./Solidity%20-%20Forced%20Ether%20%26%20On-Chain%20Privacy%20Myths.md) |
| Balance-invariant breakage (`address(this).balance == tracked total`) | [Solidity - Forced Ether & On-Chain Privacy Myths](./Solidity%20-%20Forced%20Ether%20%26%20On-Chain%20Privacy%20Myths.md) |
| `private` ≠ hidden — raw storage is publicly readable (`getStorageAt`) | [Solidity - Forced Ether & On-Chain Privacy Myths](./Solidity%20-%20Forced%20Ether%20%26%20On-Chain%20Privacy%20Myths.md) |
| Worked example: locating a packed `private` slot and extracting it via `bytesN` truncation | [Solidity - Forced Ether & On-Chain Privacy Myths](./Solidity%20-%20Forced%20Ether%20%26%20On-Chain%20Privacy%20Myths.md) |
| Ethernaut Privacy exploit (slot math + `bytes32`→`bytes16` truncation) | [Solidity - Forced Ether & On-Chain Privacy Myths](./Solidity%20-%20Forced%20Ether%20%26%20On-Chain%20Privacy%20Myths.md) |
| On-chain secrecy alternatives (commit-reveal, off-chain storage, ZK proofs) | [Solidity - Forced Ether & On-Chain Privacy Myths](./Solidity%20-%20Forced%20Ether%20%26%20On-Chain%20Privacy%20Myths.md) |
| DoS via failed external call (push-payment gating state update) | [Solidity - Denial of Service via External Calls](./Solidity%20-%20Denial%20of%20Service%20via%20External%20Calls.md) |
| Pull-over-push payment pattern | [Solidity - Denial of Service via External Calls](./Solidity%20-%20Denial%20of%20Service%20via%20External%20Calls.md) |
