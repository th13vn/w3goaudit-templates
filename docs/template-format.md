# WQL Template Format Reference

A WQL (w3goaudit Query Language) template is a single YAML file describing one
vulnerability pattern. The `w3goaudit` engine loads every `*.yaml` under the
template home, evaluates each rule against the parsed Solidity AST, and emits a
finding for every match.

A template has exactly two top-level blocks:

```yaml
meta:    # metadata about the finding
query:   # how the engine decides what to flag
```

---

## 1. The `meta` block

`meta` carries everything the reporter needs and everything a triager reads.

```yaml
meta:
  id: CRITICAL-DELEGATECALL-USER-INPUT                 # stable, unique identifier
  title: Delegatecall to User-Controlled Address
  severity: CRITICAL                # CRITICAL | HIGH | MEDIUM | LOW | INFO
  confidence: HIGH                  # HIGH | MEDIUM | LOW
  description: >
    A delegatecall is executed with a target address that comes from a
    function parameter (user-controlled). An attacker can pass a malicious
    contract address, executing arbitrary code in this contract's context.
  recommendation: >
    Never delegatecall to user-supplied addresses. Maintain a whitelist of
    trusted implementation addresses, or use a timelocked admin role.
  references:                       # optional — SWC / EIP / advisory links
    - https://swcregistry.io/docs/SWC-112
```

| Field | Required | Purpose |
| --- | --- | --- |
| `id` | yes | Unique key in the form `<SEVERITY>-<FILENAME>` (the upper-cased file name, e.g. `CRITICAL-DELEGATECALL-USER-INPUT`); appears in every finding and in `--include`/`--exclude` globs. |
| `title` | yes | One-line summary shown in reports. |
| `severity` | yes | Impact bucket — see [severity taxonomy](./severity-taxonomy.md). |
| `confidence` | yes | How precise the rule is — see [severity taxonomy](./severity-taxonomy.md). |
| `description` | recommended | What the pattern is and the risk it represents. |
| `recommendation` | recommended | How to fix it. |
| `references` | optional | External advisories / SWC entries / best-practice docs. The engine propagates these to every finding; all formatters surface them. |
| `fix` | optional | Short one-line fix summary, surfaced by formatters alongside `recommendation`. |

> **Classification (`cwe` / `owasp`):** the engine does **not** currently parse
> or emit dedicated `cwe`/`owasp` meta fields — they are silently ignored. Until
> they are supported, record any CWE / OWASP SC Top 10 classification as a
> `references` entry. The [severity taxonomy](./severity-taxonomy.md) is an
> authoring guide for *choosing* severity/confidence, not a list of emitted
> fields.

---

## 2. The `query` block

`query` holds the matching logic. A simple query has three parts:

```yaml
query:
  select: delegatecall   # (optional) the block kind whose match anchors the finding
  from: entry_function   # scope to search (default: entry_function)
  where:                 # list of matchers; siblings are implicit AND
    - ...
```

`select`/`from`/`where` replace the older `scope`/`filter`/`match` split: a
single flat `where` list carries both the function/contract preconditions and
the AST code shapes, and each matcher is evaluated at the right layer
automatically. See [§5](#5-composition--and--or) for the multi-query
`and:` / `or:` forms.

### 2.1 `select` — the finding anchor

`select` names the single block kind ([§3](#3-block-kind-catalog)) whose matched
node becomes the finding's anchor (`PrimaryAST`), e.g. `select: delegatecall`.

`select` is **optional**. When omitted, `where` must already contain an
actionable AST-level match evaluated at the scope root — a top-level
`sequence:`, a root `has:`/`any:`/`and:` structure, or (with `from: source`) a
single top-level `regex:`. A context-only `where` (just `func_name:`,
`modifier:`, or a `preset:`) cannot anchor a finding and is a load error.
A list `select: [a, b]` is rejected — use an `or:` composition instead.

### 2.2 `from` — scope

Selects which program units the rule iterates over.

| `from` | Meaning |
| --- | --- |
| `entry_function` | Public/external state-mutating functions of deployable contracts; follows internal callees. **Default** when `from` is omitted. |
| `function` | Every function everywhere. |
| `contract` / `library` / `abstract` | Contract-kind-filtered contract scope. |
| `main_contract` | Deployable contracts, evaluated against a synthetic contract AST built from the linearized inheritance chain (so one rule can span local and inherited functions). |
| `any_contract` | Every contract/interface/library. |
| `source` | Raw source-file text — regex-only (`where` holds one top-level `regex:`). |

### 2.3 `where` — the matcher grammar

`where` is a list of one-key matcher maps. Sibling items are implicit **AND**;
matchers nest uniformly (a matcher's value may itself be a matcher map), and the
same forms are legal at any depth. Empty matcher bodies are load errors.

**Leaf matchers**

- **`block: <kind>`** — node is of a given block kind or semantic group
  ([§3](#3-block-kind-catalog)), e.g. `delegatecall`, `eth_transfer`,
  `state_write`, `identifier`.
- **`name: <regex>`** — identifier/call/function name (unanchored Go regex;
  double-escape metacharacters: `"tx\\.origin"`).
- **`preset: <name>`** — a named safety property ([§4](#4-presets)).
- **`regex: <pattern>`** — raw text, line-scoped to the enclosing selected block.
- **`tainted: <source>`** — value traces to `parameter`, `state_var`,
  `local_var`, `sender`, or `user_controlled`.
- **`<attribute>: <value>`** — a semantic attribute in bare-key form, e.g.
  `visibility: public,external`, `operator: "=="`, `receiver: true`,
  `is_state_var: true`, `receiver_name: ^this$`.

**Structural & logic matchers**

- **`has: <matcher>`** — some descendant matches. **`in: <matcher>`** — some
  ancestor matches.
- **`arg.N: <matcher>`** — the Nth (0-based) call argument matches.
  **`arg.any: <matcher>`** — some positional argument matches (repeated
  `arg.any` siblings are independent existential checks).
- **`sequence: [<matcher>, ...]`** — ordered, same-execution-path occurrence
  (e.g. external call *before* a state write — the classic reentrancy shape).
- **`guarded_by: <matcher>`** — an inline guard or exact applied modifier
  declaration matches.
- **`left:` / `right:`** — operands of a binary/assignment/member node.
- **`func_name:` / `modifier:` / `version:` / `has_param:` / `base:`** —
  function/contract preconditions.
- **`any: [...]`** (OR), **`and: [...]`** (explicit AND grouping),
  **`not: <matcher>`** (negation). Repeated sibling `not:` items are independent
  predicates: `not: A` plus `not: B` means `(not A) and (not B)`.

Presets and `not:` are how you express a property's **absence** — e.g.
`not: { preset: access_controlled }` matches functions lacking privileged
access control. Each preset returns true when its property is *present*, so it
reads naturally either asserted or negated.

---

## 3. Block-kind catalog

Common kinds usable in `select:` and as `block: <kind>` in `where`:

| Group | Kinds |
| --- | --- |
| Calls | `call`, `external_call`, `internal_call`, `delegatecall`, `staticcall`, `lowlevel_call`, `create`, `eth_transfer`, `selfdestruct`, `builtin_transfer`, `builtin_send` |
| Guards | `guard`, `require`, `assert`, `revert` |
| State | `state_write` (assign / storage-array `push`·`pop` / state `delete`·`++`·`--` / `asm.sstore`), `state_read` |
| Statements | `assign`, `if`, `loop`, `unchecked`, `return`, `emit`, `try_catch`, `block` |
| Expressions | `identifier`, `literal`, `binary`, `unary`, `member`, `index`, `ternary`, `tuple` |
| Declarations | `function`, `contract`, `modifier`, `variable`, `parameter` |
| Assembly | `asm`, `asm_sstore`, `asm_sload`, `asm_delegatecall`, `asm_call`, `asm_staticcall`, `asm_create`, `asm_selfdestruct`, `asm_revert`, `asm_return` |

Dot-notation groups match their family (`call` covers all `call.*`). The full
catalog, attribute list, and taint model live in the engine reference at
[`w3goaudit/docs/wql-syntax.md`](https://github.com/th13vn/w3goaudit/blob/main/docs/wql-syntax.md).

---

## 4. Presets

Each preset returns true when its named safety property is **present**, so use
`not: { preset: ... }` to match the vulnerable (absent) case.

| Preset (property = true) | Present when… |
| --- | --- |
| `access_controlled` | Privileged access control is present — an owner/admin/role modifier, an internal auth-helper call, or a `msg.sender`/`tx.origin` guard against a hardcoded/storage owner (checked recursively through internal/inherited calls). `require(from == msg.sender)` is item-ownership self-scoping, **not** privileged auth, so it does not clear this preset. |
| `caller_checked` | Either the function is `access_controlled`, **or** it self-scopes the caller — binds a sensitive argument to `msg.sender` (`require(from == msg.sender)`), including the item-ownership form `ownerOf(tokenId) == msg.sender`. Use where binding a sensitive arg to the caller is a valid mitigation (e.g. arbitrary `transferFrom`). |
| `reentrancy_guarded` | A reentrancy-guard modifier (`nonReentrant`/`lock`/`mutex`/etc.) is present. |

Unknown preset names are rejected at load. Presets compose anywhere in `where`,
including under `any:` / `not:`.

---

## 5. Composition — `and:` / `or:`

Instead of `select`/`from`/`where`, `query` may hold a one-level composition:

- **`or:`** — a list of branches, each a full `{select, from, where}` query. The
  engine runs each branch and unions the findings (deduplicating the same precise
  site). Use for "shape A **or** shape B" detectors.
- **`and:`** — a list of labeled branches that must all match in one shared
  scope; each branch may carry a `label:` naming its site in the finding's
  related context. Use for multi-site combination findings in one contract.

`and:` and `or:` are mutually exclusive and cannot be combined with a top-level
`select`/`from`/`where` in the same `query`.

---

## 6. Worked example

The reentrancy rule combines a function-level precondition with an ordered AST
pattern in one flat `where`:

```yaml
meta:
  id: HIGH-REENTRANCY-PATTERN
  title: Potential Reentrancy - State Variable Modification
  severity: HIGH
  confidence: MEDIUM
  description: >
    Detects functions where external calls occur before state variable updates.
    Recursively traces through internal function calls to find external calls.
  recommendation: >
    Apply the Check-Effects-Interactions pattern (update state BEFORE external
    calls) or use a ReentrancyGuard modifier.
  references:
    - https://swcregistry.io/docs/SWC-107

query:
  from: entry_function
  where:
    # true when the function has NO reentrancy guard — exactly the case to scan
    - not: { preset: reentrancy_guarded }
    # external call followed by a state write
    - sequence:
        - any:
            - block: eth_transfer   # .transfer / .send / .call{value:} / raw .call
            - block: delegatecall   # arbitrary code execution in this context
        - block: state_write
```

How the engine reads this:

1. `from: entry_function` → iterate externally reachable functions.
2. `not: { preset: reentrancy_guarded }` → keep only functions with no guard.
3. `sequence` → find an ETH-bearing/raw call **or** a delegatecall that occurs
   **before** a state write. Its first step is the finding anchor.
4. On a match, emit a finding carrying the `meta` (severity, description,
   recommendation, references).

---

## 7. Authoring checklist

- [ ] Unique, stable `id` of the form `<SEVERITY>-<FILENAME>` (upper-cased file name); the file lives in the matching `official/<severity>/` folder.
- [ ] `severity` and `confidence` justified and consistent with peers.
- [ ] Preconditions and code shapes both live in one flat `where` (or an `and:`/`or:` composition); the engine routes each matcher to the right layer.
- [ ] `select` present, or an actionable root matcher (`sequence:`/`has:`/`any:`/root `regex:`) so the finding can anchor.
- [ ] Rule fires on a vulnerable fixture and is silent on a safe one.
- [ ] `references` set where a canonical advisory exists (SWC / EIP / Slither).
- [ ] Non-obvious FP/TP trade-offs documented in YAML comments.
