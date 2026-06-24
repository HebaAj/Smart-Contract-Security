# Smart Contract Security

A structured knowledge base covering Solidity fundamentals and smart contract security patterns — built through hands-on work with [CryptoZombies](https://cryptozombies.io) and [Ethernaut](https://ethernaut.openzeppelin.com).

Each note prioritises **understanding over memorisation**: language features are explained alongside the security implications they carry, and every attack pattern traces back to the specific assumption it breaks.

---

## Topics Covered

### Language Fundamentals
Core Solidity before the security layer.

| Note | Topics |
|------|--------|
| [Core Data Types & Structures](./Solidity%20-%20Core%20Data%20Types%20%26%20Structures.md) | `uint` family, structs, arrays, mappings, strings, time |
| [Data Location & Gas Model](./Solidity%20-%20Data%20Location%20%26%20Gas%20Model.md) | storage/memory/calldata, gas cost model, struct packing, rebuild-vs-maintain pattern |
| [Functions, Visibility & Modifiers](./Solidity%20-%20Functions%2C%20Visibility%20%26%20Modifiers.md) | public/private/internal/external, view/pure, constructors, modifiers, events |
| [Hashing, Encoding & Type Conversion](./Solidity%20-%20Hashing%2C%20Encoding%20%26%20Type%20Conversion.md) | `keccak256`, `abi.encodePacked` vs `encode`, truncation direction, XOR self-inverse |
| [Inheritance & Contract Interaction](./Solidity%20-%20Inheritance%20%26%20Contract%20Interaction.md) | inheritance, `Ownable` pattern, `msg.sender`/`require`, interfaces |
| [Payable Functions & Ether Transfer](./Solidity%20-%20Payable%20Functions%20%26%20Ether%20Transfer.md) | `payable`, `msg.value`, withdraw pattern, `.transfer`/`.send`/`.call`, reentrancy |
| [ABI Encoding, Selectors & Console Interaction](./Solidity%20-%20ABI%20Encoding%2C%20Selectors%20%26%20Console%20Interaction.md) | function selectors, calldata construction, web3 console workflow |
| [Events as Storage & On-Chain Interaction Model](./Solidity%20-%20Events%20as%20Storage%20%26%20On-Chain%20Interaction%20Model.md) | events vs storage gas tradeoff, `indexed` fields, call vs send |
| [Token Standards (ERC-20 & ERC-721)](./Solidity%20-%20Token%20Standards%20%28ERC-20%20%26%20ERC-721%29.md) | fungible vs non-fungible, standard interfaces, two-step approval flow |
| [Integer Overflow & SafeMath](./Solidity%20-%20Integer%20Overflow%20%26%20SafeMath.md) | silent wrapping pre-0.8, SafeMath, checked arithmetic in ≥0.8, `unchecked` blocks |

### Security Patterns & Attacks
Each note covers the vulnerability, the exploit, and the fix.

| Note | Topics |
|------|--------|
| [delegatecall & Execution Context](./Solidity%20-%20delegatecall%20%26%20Execution%20Context.md) | call vs delegatecall vs staticcall, storage slot alignment, proxy pattern, Delegation exploit |
| [Access Control Bypass via Execution Context](./Solidity%20-%20Access%20Control%20Bypass%20via%20Execution%20Context.md) | `tx.origin` phishing, `extcodesize` constructor bypass, `gasleft()` brute-forcing |
| [Trusting External Calls & Interface Assumptions](./Solidity%20-%20Trusting%20External%20Calls%20%26%20Interface%20Assumptions.md) | interfaces as shape-only promises, non-idempotent external calls, Elevator exploit |
| [Forced Ether & On-Chain Privacy Myths](./Solidity%20-%20Forced%20Ether%20%26%20On-Chain%20Privacy%20Myths.md) | `selfdestruct` forced ETH, balance invariant breakage, `private` storage transparency |
| [Denial of Service via External Calls](./Solidity%20-%20Denial%20of%20Service%20via%20External%20Calls.md) | push-payment DoS, pull-over-push fix |
| [Randomness Attacks & Mitigations](./Solidity%20-%20Randomness%20Attacks%20%26%20Mitigations.md) | block-data predictability, simulate-before-publish, Chainlink VRF, commit-reveal |

---

## Navigation

The [Smart Contract Security Notes MOC](./Smart%20Contract%20Security%20Notes%20MOC.md) lists every note and provides a full concept quick-reference table.

---

## Using These Notes

**On GitHub:** all cross-references are clickable links. Browse any note and follow the "See also" links at the bottom to navigate related topics.

**In Obsidian:** clone the repository and open the folder as a vault. Standard markdown links work as-is, or convert to wiki-link syntax with a find-and-replace.
