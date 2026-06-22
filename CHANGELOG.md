# Changelog

All notable changes to the official w3goaudit template pack are documented here.
This project adheres to [Semantic Versioning](https://semver.org/).

## [v1.0.1] - 2026-06-22

Maintenance release: repository reorganization, a template-ID rename, and
documentation-accuracy fixes. **No detection logic changed** — every rule
matches exactly as it did in v1.0.0, and a bare `w3goaudit ./contracts/` scan
produces the same findings (under the new IDs).

### Changed
- **`official/` is now split by severity** into `critical/` (2), `high/` (15),
  and `medium/` (8) subfolders. The engine loads templates recursively, so the
  default scan and `--template official/` are unaffected.
- **All 25 template IDs renamed** to the form `<SEVERITY>-<FILENAME>` (the
  upper-cased file name), replacing the old `SEC-<CATEGORY>-NNN` scheme so the ID
  is self-describing and stays in sync with the file.

  > ⚠️ **Breaking for `--include` / `--exclude`:** globs referencing old `SEC-*`
  > IDs must be updated. Full mapping:

  | Old ID | New ID |
  | --- | --- |
  | `SEC-DELEG-001` | `CRITICAL-DELEGATECALL-USER-INPUT` |
  | `SEC-DEST-001` | `CRITICAL-SELFDESTRUCT-UNPROTECTED` |
  | `SEC-CALL-001` | `HIGH-ARBITRARY-LOW-LEVEL-CALL` |
  | `SEC-ETH-001` | `HIGH-ARBITRARY-SEND-ETH` |
  | `SEC-ERC20-001` | `HIGH-ARBITRARY-TRANSFERFROM` |
  | `SEC-DELEG-002` | `HIGH-DELEGATECALL-IN-LOOP` |
  | `SEC-SIG-001` | `HIGH-ECDSA-RECOVER-MALLEABLE` |
  | `SEC-HASH-001` | `HIGH-ENCODE-PACKED-COLLISION` |
  | `SEC-MATH-003` | `HIGH-INCORRECT-EXP` |
  | `SEC-MSGVAL-001` | `HIGH-MSG-VALUE-IN-LOOP` |
  | `SEC-PROXY-001` | `HIGH-PROXY-STORAGE-COLLISION` |
  | `SEC-REENTRANCY-002` | `HIGH-REENTRANCY-BALANCE` |
  | `SEC-GEN-REENTRANCY` | `HIGH-REENTRANCY-PATTERN` |
  | `SEC-ERC20-002` | `HIGH-UNCHECKED-ERC20-TRANSFER` |
  | `SEC-UPGRADE-001` | `HIGH-UNPROTECTED-INITIALIZER` |
  | `SEC-OWNER-001` | `HIGH-UNRESTRICTED-TRANSFEROWNERSHIP` |
  | `SEC-PRNG-001` | `HIGH-WEAK-PRNG` |
  | `SEC-BOOL-001` | `MEDIUM-BOOLEAN-CST` |
  | `SEC-EQ-001` | `MEDIUM-DANGEROUS-STRICT-EQUALITY` |
  | `SEC-MATH-002` | `MEDIUM-DIVIDE-BEFORE-MULTIPLY` |
  | `SEC-TXORIGIN-001` | `MEDIUM-TX-ORIGIN-AUTH` |
  | `SEC-MATH-001` | `MEDIUM-UNCHECKED-ARITHMETIC` |
  | `SEC-LOWLEVEL-CALL-001` | `MEDIUM-UNCHECKED-LOWLEVEL-CALL` |
  | `SEC-SEND-001` | `MEDIUM-UNCHECKED-SEND` |
  | `SEC-MUTABILITY-001` | `MEDIUM-VIEW-PURE-MODIFIES-STATE` |

### Fixed (documentation)
- Removed the non-existent `cwe` / `owasp` meta fields from the docs — the engine
  parses and emits neither (`TemplateMeta` has only `id`, `title`, `severity`,
  `confidence`, `description`, `recommendation`, `references`, `fix`). Record CWE
  / OWASP classification as a `references` entry instead.
- Corrected the filter-preset reference to the three real presets
  (`unAuthenticated`, `unCheckedSender`, `unLocked`) and removed the bogus
  `view` / `pure` "presets" (those are `mutability_filter` values).
- Completed the `match` matcher reference (`inside`, `all`, `not`, `name`,
  `args`, `left` / `right`, `source_regex`) and the full `tainted_from` value set
  (`parameter`, `state_var`, `local_var`, `sender`).
- Documented the optional `fix` meta field.

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

[v1.0.1]: https://github.com/th13vn/w3goaudit-templates/releases/tag/v1.0.1
[v1.0.0]: https://github.com/th13vn/w3goaudit-templates/releases/tag/v1.0.0
