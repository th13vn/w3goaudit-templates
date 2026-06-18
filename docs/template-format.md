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
  id: SEC-DELEG-001                 # stable, unique identifier
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
  cwe: CWE-829                      # recommended
  owasp: SC02:2025                  # recommended — OWASP SC Top 10 2025 category
  references:                       # optional
    - https://swcregistry.io/docs/SWC-112
```

| Field | Required | Purpose |
| --- | --- | --- |
| `id` | yes | Unique key (`SEC-<CATEGORY>-NNN`); appears in every finding and in `--include`/`--exclude` globs. |
| `title` | yes | One-line summary shown in reports. |
| `severity` | yes | Impact bucket — see [severity taxonomy](./severity-taxonomy.md). |
| `confidence` | yes | How precise the rule is — see [severity taxonomy](./severity-taxonomy.md). |
| `description` | yes | What the pattern is and the risk it represents. |
| `recommendation` | yes | How to fix it. |
| `cwe` | recommended | CWE identifier(s). |
| `owasp` | recommended | OWASP Smart Contract Top 10 (2025) category. |
| `references` | optional | External advisories / SWC entries / best-practice docs. |

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
is most often a named **preset** that captures a function-level property:

| Preset | True when… |
| --- | --- |
| `unAuthenticated` | the function has no access-control guard |
| `unLocked` | the function lacks a reentrancy guard |
| `view`, `pure` | the declared mutability of the function |

A filter never produces a finding by itself — it only gates the `match`.

### 2.3 `match` — AST pattern

`match` is the pattern that, when found inside a filtered function, yields a
finding. It is a tree of matchers:

- **`kind:`** — match a node of a given kind, e.g. `delegatecall`,
  `eth_transfer`, `state_write`, `expr.identifier`, `call.lowlevel.call`.
- **`contains:`** — the node must contain a descendant matching the nested
  pattern (nestable).
- **`sequence:`** — an ordered list of patterns that must appear in order
  (e.g. external call *before* a state write — the classic reentrancy shape).
- **`any:`** — matches if any of the listed sub-patterns match (logical OR).
- **`attr:`** — assert node attributes, e.g. `call_receiver: true`.
- **`tainted_from:`** — dataflow constraint, e.g. `parameter` means the value
  originates from a user-controlled function parameter.

---

## 3. Worked example

The reentrancy rule combines a function-level precondition with an
ordered AST pattern:

```yaml
meta:
  id: SEC-GEN-REENTRANCY
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

- [ ] Unique, stable `id` following `SEC-<CATEGORY>-NNN`.
- [ ] `severity` and `confidence` justified and consistent with peers.
- [ ] Preconditions live in `filter`; code shapes live in `match`.
- [ ] Rule fires on a vulnerable fixture and is silent on a safe one.
- [ ] `cwe` / `owasp` (SC Top 10 2025) set where applicable.
- [ ] Non-obvious FP/TP trade-offs documented in YAML comments.
