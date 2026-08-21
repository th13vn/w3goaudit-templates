# Changelog

All notable changes to the official w3goaudit template pack are documented here.
This project adheres to [Semantic Versioning](https://semver.org/).

## [v2.1.1] - 2026-07-31

One precision fix, no rule added, removed, or renamed. Engine floor is unchanged
(`w3goaudit` >= 0.4.2) — this pack uses no new matcher.

### Fixed — `HIGH-REENTRANCY-PATTERN` no longer matches `delegatecall`

The CEI rule matched `delegatecall` alongside ETH-bearing and raw low-level calls.
Delegatecall executes the callee's bytecode in **this** contract's context — the
same storage, the same guard slot — so re-entry through it is the contract running
its own code, not attacker-injected reentry. In practice the target is a trusted,
admin-set implementation or sidecar (`proxyPaths_[CONST].delegatecall(...)`), or
literally `address(this)` (multicall).

On the 470-contract corpus this was **3 of 7** findings in that class, all noise:
CrocSwap `ProxyCaller.callSwap` / `callCrossFlag` and Eesee
`MulticallPayable.multicall`. No true positive is lost — when the delegatecall
target is attacker-influenced the bug is arbitrary code execution, strictly worse
than reentrancy, and `CRITICAL-DELEGATECALL-USER-INPUT` already reports it.
`HIGH-DELEGATECALL-IN-LOOP` and `MEDIUM-VIEW-PURE-MODIFIES-STATE` are unchanged.

The rule now reads:

```yaml
- sequence:
    - block: eth_transfer   # .transfer / .send / .call{value:} / raw .call / asm.call
    - block: state_write
```

### Note — one companion fix ships in the engine, not here

`HIGH-UNPROTECTED-INITIALIZER` reported Axelar's `InterchainToken.init()` despite
its `if (_isInitialized()) revert AlreadyInitialized();` guard, because the
`initializer_guarded` preset did not treat an inline-assembly
`sstore(INITIALIZED_SLOT, true)` as setting the flag. That is engine-side and is
fixed in **`w3goaudit` v0.4.3**; `--update-templates` cannot deliver it. Upgrade
the binary as well.

## [v2.1.0] - 2026-07-29

Adds 14 detectors (25 → 39) and tightens 5 existing ones after auditing every
finding a 470-file corpus of verified block-explorer sources produced (Kyber,
Euler, CrocSwap, OpenZeppelin, Uniswap v4).

> **Requires `w3goaudit` >= 0.4.2.** The pack uses the `receiver:` matcher and the
> `initializer_guarded` preset. An older engine **fails to load the entire pack**
> (`unknown preset "initializer_guarded"`) rather than skipping the affected rules,
> so upgrade the binary before running `--update-templates`. This is a minor bump
> because no rule was removed or renamed, but treat the engine floor as hard.

### Added — 14 detectors promoted from the benchmark lane

Generic, benchmark-validated rules that previously existed only in the engine
repository's dev-only harness, so a released binary never ran them. Each ships with
vulnerable **and** safe fixtures upstream.

- Read-only reentrancy: `HIGH-READONLY-REENTRANCY-CURVE`,
  `HIGH-READONLY-REENTRANCY-BALANCER`
- AMM integration: `HIGH-NO-SLIPPAGE-CHECK`,
  `HIGH-UNISWAP-CALLBACK-UNPROTECTED`, `HIGH-UNISWAP-V4-HOOK-UNPROTECTED`
- Access control: `HIGH-ORACLE-PRICE-UPDATE-UNRESTRICTED`, `HIGH-ERC20-PUBLIC-BURN`
- Callback reentrancy: `HIGH-ERC721-CALLBACK-REENTRANCY`,
  `HIGH-ERC777-CALLBACK-REENTRANCY`
- Oracles: `MEDIUM-STALE-ORACLE-DATA`,
  `MEDIUM-DEPRECATED-CHAINLINK-LATESTANSWER`, `MEDIUM-ORACLE-LP-SPOT-PRICE`,
  `MEDIUM-ORACLE-CURVE-SPOT-PRICE`
- Source integrity: `MEDIUM-BIDI-CHARACTERS` (Trojan Source)

A fifteenth candidate — a generic "external call then state write" reentrancy rule —
was evaluated and rejected: it fired on 36 sites across the upstream fixtures, 17 of
them deliberate safe controls, because pull-then-credit is the correct ERC20 deposit
shape. The precise signal it reached for is already covered by the two
callback-reentrancy rules above.

### Changed — 5 detectors tightened

Each of these reported real, audited production code as vulnerable.

- `CRITICAL-DELEGATECALL-USER-INPUT` — constrains the call's **direct** receiver via
  the new `receiver:` matcher. `has: {receiver: true, …}` searched every descendant,
  so the receiver of a nested call inside the encoded arguments satisfied it:
  CrocSwapDex's `proxyPaths_[CONST].delegatecall(abi.encodeWithSignature(…,
  curve.pullPriceTick(), …))` was CRITICAL because `curve` is caller-derived, not
  because the target was.
- `HIGH-ARBITRARY-LOW-LEVEL-CALL` — same direct-receiver fix, plus the caller must
  control the **selector**. Contract-built calldata with a pinned selector
  (`abi.encodeWithSelector(IERC20.transferFrom.selector, …)` — Uniswap's
  `TransferHelper`) lets the caller choose only the arguments, and was reported in
  four separate projects.
- `HIGH-ECDSA-RECOVER-MALLEABLE` — the replay guard must be keyed by the
  **signature**. Any caller-derived index matched, so `claimed[battleId] = true` — an
  id-keyed flag, i.e. the correct pattern — was reported.
- `HIGH-WEAK-PRNG` — excludes a modulo by a power of two.
  `uint32(block.timestamp % 2**32)` is Uniswap V2's slot-packing truncation, not a
  random draw.
- `MEDIUM-UNCHECKED-ARITHMETIC` — anchors on the **operation** rather than the
  `unchecked` block (a gas-optimized whole-function wrapper produced one
  unactionable finding for an entire body), and covers only `*`, `-`, and `+` when an
  operand is caller-controlled. `/` cannot overflow; counter and nonce increments
  cannot be driven to the type maximum.
- `MEDIUM-BOOLEAN-CST` — only the constant-folding shapes (`a && false`,
  `if (true)`). `x == false` is a verbose `!x`: a style preference, and the idiomatic
  mutex spelling.
- `MEDIUM-DANGEROUS-STRICT-EQUALITY` — block-clock comparisons only when the other
  operand is caller-controlled, so Aave's same-block cache short-circuit
  (`if (reserve.lastUpdateTimestamp == block.timestamp) return cachedIndex;`) is not
  reported.
- `HIGH-UNPROTECTED-INITIALIZER` — uses the new `initializer_guarded` preset, which
  proves a one-time guard from the body (an enforcing check on a state flag that the
  function, a helper, or a sibling such as the constructor sets). It previously knew
  only the OpenZeppelin `initializer` modifier and a guard literally naming
  `_initializersDisabled`, so Euler's `if (initialized) revert; initialized = true;`
  and Axelar's split `if (_isInitialized()) revert; _initialize();` were both
  reported.

Net effect on that corpus, combining these with the engine-side fixes in
`w3goaudit` v0.4.2: **103 → 44 findings**, with the competitive benchmark unchanged
at 109 true positives and 100% recall.

## [v2.0.0] - 2026-07-20

Breaking **WQL schema** migration to the canonical query language shipped with
`w3goaudit` v0.4. **No detection logic changed** — all 25 rules match exactly as
they did in v1.0.1 (verified: a `w3goaudit test-data/security` scan produces the
same 110 findings under the same IDs with this pack as with the engine's
embedded pack). Requires `w3goaudit` **v0.4.0+**; older engines cannot load
these templates.

### Changed
- **Every template migrated from the old `query: {scope, filter, match}` form to
  the canonical `query: {select, from, where}` form** (or an `and:` / `or:`
  composition). The two-block precondition/AST split is replaced by one flat
  `where` matcher list; the engine routes each matcher to the right layer.
  - `scope: entrypoint` → `from: entry_function` (and the other `from` scopes).
  - `filter:`/`match:` merged into `where:`.
  - `kind:` → `block:`; `contains:` → `has:`; `inside:` → `in:`;
    `tainted_from:` → `tainted:` (now also accepts `user_controlled`);
    `attr: {call_receiver: true}` → `receiver: true`; `all:` → `and:`.
  - **Presets renamed to intuitive polarity** and now express *presence* of a
    safety property, so the vulnerable case negates them:
    `unAuthenticated` → `not: {preset: access_controlled}`,
    `unCheckedSender` → `not: {preset: caller_checked}`,
    `unLocked` → `not: {preset: reentrancy_guarded}`.
- **Template IDs are unchanged** (`<SEVERITY>-<FILENAME>`), so `--include` /
  `--exclude` globs continue to work.
- Documentation (`README.md`, `CONTRIBUTING.md`, `docs/template-format.md`)
  rewritten for the canonical grammar; added a repository `AI.md` map.

### Migration
- Run `w3goaudit --update-templates` on **v0.4.0+** to pull this release. On an
  older engine, keep pinning the `v1.0.1` tag until you upgrade.

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

  | Old ID                  | New ID                                |
  | ----------------------- | ------------------------------------- |
  | `SEC-DELEG-001`         | `CRITICAL-DELEGATECALL-USER-INPUT`    |
  | `SEC-DEST-001`          | `CRITICAL-SELFDESTRUCT-UNPROTECTED`   |
  | `SEC-CALL-001`          | `HIGH-ARBITRARY-LOW-LEVEL-CALL`       |
  | `SEC-ETH-001`           | `HIGH-ARBITRARY-SEND-ETH`             |
  | `SEC-ERC20-001`         | `HIGH-ARBITRARY-TRANSFERFROM`         |
  | `SEC-DELEG-002`         | `HIGH-DELEGATECALL-IN-LOOP`           |
  | `SEC-SIG-001`           | `HIGH-ECDSA-RECOVER-MALLEABLE`        |
  | `SEC-HASH-001`          | `HIGH-ENCODE-PACKED-COLLISION`        |
  | `SEC-MATH-003`          | `HIGH-INCORRECT-EXP`                  |
  | `SEC-MSGVAL-001`        | `HIGH-MSG-VALUE-IN-LOOP`              |
  | `SEC-PROXY-001`         | `HIGH-PROXY-STORAGE-COLLISION`        |
  | `SEC-REENTRANCY-002`    | `HIGH-REENTRANCY-BALANCE`             |
  | `SEC-GEN-REENTRANCY`    | `HIGH-REENTRANCY-PATTERN`             |
  | `SEC-ERC20-002`         | `HIGH-UNCHECKED-ERC20-TRANSFER`       |
  | `SEC-UPGRADE-001`       | `HIGH-UNPROTECTED-INITIALIZER`        |
  | `SEC-OWNER-001`         | `HIGH-UNRESTRICTED-TRANSFEROWNERSHIP` |
  | `SEC-PRNG-001`          | `HIGH-WEAK-PRNG`                      |
  | `SEC-BOOL-001`          | `MEDIUM-BOOLEAN-CST`                  |
  | `SEC-EQ-001`            | `MEDIUM-DANGEROUS-STRICT-EQUALITY`    |
  | `SEC-MATH-002`          | `MEDIUM-DIVIDE-BEFORE-MULTIPLY`       |
  | `SEC-TXORIGIN-001`      | `MEDIUM-TX-ORIGIN-AUTH`               |
  | `SEC-MATH-001`          | `MEDIUM-UNCHECKED-ARITHMETIC`         |
  | `SEC-LOWLEVEL-CALL-001` | `MEDIUM-UNCHECKED-LOWLEVEL-CALL`      |
  | `SEC-SEND-001`          | `MEDIUM-UNCHECKED-SEND`               |
  | `SEC-MUTABILITY-001`    | `MEDIUM-VIEW-PURE-MODIFIES-STATE`     |

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

[v2.0.0]: https://github.com/th13vn/w3goaudit-templates/releases/tag/v2.0.0
[v1.0.1]: https://github.com/th13vn/w3goaudit-templates/releases/tag/v1.0.1
[v1.0.0]: https://github.com/th13vn/w3goaudit-templates/releases/tag/v1.0.0
