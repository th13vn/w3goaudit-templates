# Changelog

All notable changes to the official w3goaudit template pack are documented here.
This project adheres to [Semantic Versioning](https://semver.org/).

## [v1.0.0] - 2026-06-18

Initial public release of the official WQL template pack — **25 templates**.

### Access control & ownership
- `SEC-DEST-001` — Unprotected selfdestruct (CRITICAL)
- `SEC-ETH-001` — Unprotected ETH Withdrawal (HIGH)
- `SEC-UPGRADE-001` — Unprotected Initializer (Anyone Can Become Owner) (HIGH)
- `SEC-OWNER-001` — Unrestricted transferOwnership (HIGH)
- `SEC-TXORIGIN-001` — tx.origin Used for Authentication (MEDIUM)

### Delegatecall & low-level calls
- `SEC-DELEG-001` — Delegatecall to User-Controlled Address (CRITICAL)
- `SEC-DELEG-002` — Delegatecall Inside a Loop (HIGH)
- `SEC-CALL-001` — Arbitrary Low-Level Call (HIGH)
- `SEC-LOWLEVEL-CALL-001` — Unchecked Low-Level Call Return Value (MEDIUM)
- `SEC-SEND-001` — Unchecked send() Return Value (MEDIUM)

### Reentrancy
- `SEC-GEN-REENTRANCY` — Potential Reentrancy: State Variable Modification (HIGH)
- `SEC-REENTRANCY-002` — Reentrancy via balanceOf Delta (HIGH)

### Tokens / ERC20
- `SEC-ERC20-001` — Arbitrary transferFrom Call (HIGH)
- `SEC-ERC20-002` — Unchecked ERC20 transfer / transferFrom Return Value (HIGH)

### Arithmetic
- `SEC-MATH-001` — Arithmetic in Unchecked Block (MEDIUM)
- `SEC-MATH-002` — Division Before Multiplication (MEDIUM)
- `SEC-MATH-003` — Suspicious Use of `^` (Bitwise XOR, Likely Meant `**`) (HIGH)
- `SEC-EQ-001` — Dangerous Strict Equality on Externally-Influenced Balance (MEDIUM)

### Cryptography & randomness
- `SEC-PRNG-001` — Weak Randomness from Block Variables (HIGH)
- `SEC-SIG-001` — Signature Malleability: Raw Signature Bytes as State Key (HIGH)
- `SEC-HASH-001` — Hash Collision via abi.encodePacked (HIGH)

### Denial of service
- `SEC-MSGVAL-001` — msg.value Used Inside a Loop (HIGH)

### Upgradeability
- `SEC-PROXY-001` — Proxy Storage Layout Collision (HIGH)

### Code correctness
- `SEC-BOOL-001` — Boolean Constant Misuse (MEDIUM)
- `SEC-MUTABILITY-001` — view/pure Function Modifies State (MEDIUM)

[v1.0.0]: https://github.com/th13vn/w3goaudit-templates/releases/tag/v1.0.0
