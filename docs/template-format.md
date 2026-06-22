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

`query` holds the matching logic. It has three parts:

```yaml
query:
  scope: entrypoint     # what units to evaluate
  filter:               # function/contract preconditions
    ...
  match:                # AST pattern that produces the finding
    ...
```

### 2.1 `scope`

Selects which program units the rule iterates over. The common value is
`entrypoint` (externally reachable functions). The scope bounds the cost of the
rule and the meaning of the filter.

### 2.2 `filter` — preconditions

`filter` decides **which** functions/contracts in scope are even considered. It
is most often a named **preset** that captures a function-level property. Every
preset returns `true` for the **vulnerable** case, so use it *without* a `not:`
wrapper. There are exactly three:

| Preset | True when… |
| --- | --- |
| `unAuthenticated` | the function has no privileged access-control guard (`require(from == msg.sender)` is *not* privileged auth, so it does not clear this preset) |
| `unCheckedSender` | like `unAuthenticated`, but **also** clears functions that self-scope the caller (`require(from == msg.sender)`) — use where binding a sensitive arg to the caller is a valid mitigation (e.g. arbitrary `transferFrom`) |
| `unLocked` | the function lacks a reentrancy guard |

Unknown preset names are rejected at template load. Note that `view` / `pure`
are **not** presets — match function mutability with
`mutability_filter: view,pure` instead. Other function/contract preconditions
(`modifier`, `func_name`, `visibility_filter`, `mutability_filter`, `extends`,
`version`, `has_param`, `has_guard`, `source_regex`, and `not:`) live in `filter`
too.

A filter never produces a finding by itself — it only gates the `match`.

### 2.3 `match` — AST pattern

`match` is the pattern that, when found inside a filtered function, yields a
finding. It is a tree of matchers. When several fields are set in one rule the
default logic is **AND** (all must match).

Node / value matchers:

- **`kind:`** — match a node of a given kind or semantic group, e.g.
  `delegatecall`, `eth_transfer`, `state_write`, `expr.identifier`,
  `call.lowlevel.call`. Dot-notation prefixes match (`call` matches all `call.*`).
- **`name:`** — match a node's name/value as a Go regex (quote and double-escape
  metacharacters: `"tx\\.origin"`).
- **`attr:`** — assert node attributes, e.g. `call_receiver: true`,
  `subtype: bool`, `operator: "=="`. Inline attribute keys also work.
- **`tainted_from:`** — dataflow constraint. Accepts exactly `parameter`,
  `state_var`, `local_var`, or `sender`; any other value is rejected at load.
- **`args:`** (a.k.a. `arg.N`) — match a call's positional arguments by index.
- **`left:` / `right:`** — match the direct operands of a binary/member node.
- **`source_regex:`** — match raw source text (scope-aware).

Traversal & logic combinators:

- **`contains:`** — the node must contain a descendant matching the nested
  pattern (nestable).
- **`inside:`** — the node must have an ancestor matching the nested pattern.
- **`sequence:`** — an ordered list of patterns that must appear in source order
  (e.g. external call *before* a state write — the classic reentrancy shape).
- **`all:`** — every sub-pattern must match (explicit AND).
- **`any:`** — matches if any sub-pattern matches (logical OR).
- **`not:`** — negate the nested pattern.

---

## 3. Worked example

The reentrancy rule combines a function-level precondition with an
ordered AST pattern:

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
  scope: entrypoint

  filter:
    # unLocked is true when the function has no reentrancy guard — exactly the
    # case we want to scan. No `not:` wrapper needed.
    preset: unLocked

  match:
    # external call followed by a state write
    sequence:
      - any:
          - kind: eth_transfer   # .transfer / .send / .call{value:} / raw .call
          - kind: delegatecall   # arbitrary code execution in this context
      - kind: state_write
```

How the engine reads this:

1. `scope: entrypoint` → iterate externally reachable functions.
2. `filter: preset: unLocked` → keep only functions with no reentrancy guard.
3. `match: sequence` → in each kept function, find an ETH-bearing/raw call **or**
   a delegatecall that occurs **before** a state write.
4. On a match, emit a finding carrying the `meta` (HIGH / MEDIUM, with the
   description and recommendation).

---

## 4. Authoring checklist

- [ ] Unique, stable `id` of the form `<SEVERITY>-<FILENAME>` (upper-cased file name); the file lives in the matching `official/<severity>/` folder.
- [ ] `severity` and `confidence` justified and consistent with peers.
- [ ] Preconditions live in `filter`; code shapes live in `match`.
- [ ] Rule fires on a vulnerable fixture and is silent on a safe one.
- [ ] `references` set where a canonical advisory exists (SWC / EIP / Slither).
- [ ] Non-obvious FP/TP trade-offs documented in YAML comments.
