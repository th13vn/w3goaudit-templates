# WQL Template Format

WQL is W3GoAudit's deterministic rule language. A template describes one
vulnerability mechanism and produces findings from Solidity semantic and AST
evidence. Pack `v3.0.0` uses the WQL surface in W3GoAudit `v0.6.0` or newer.

This guide is the approachable pack-facing reference. The complete engine
grammar lives in
[`w3goaudit/docs/wql-syntax.md`](https://github.com/th13vn/w3goaudit/blob/main/docs/wql-syntax.md).

## 1. Document Shape

A file has exactly two top-level blocks:

```yaml
meta:
  id: HIGH-EXAMPLE
  title: Example vulnerability mechanism
  severity: HIGH
  confidence: MEDIUM
  description: >
    Explain the condition the query proves and the resulting risk.
  recommendation: >
    Explain a concrete mitigation.

query:
  from: entry_function
  select: external_call
  where:
    - name: ^dangerousCall$
```

Unknown keys, YAML merge keys, empty matcher bodies, invalid regexes, unknown
block kinds, and unsupported presets fail at load time. The scanner does not
silently skip invalid templates unless the user explicitly enables lenient
loading.

## 2. Metadata

| Field | Official pack | Meaning |
|---|---|---|
| `id` | Required | Stable `<SEVERITY>-<FILENAME>` identifier |
| `title` | Required | Concise sentence-case summary |
| `severity` | Required | `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, or `INFO` |
| `confidence` | Required | `HIGH`, `MEDIUM`, or `LOW` |
| `description` | Required | The exact proven condition and risk |
| `recommendation` | Required | Actionable mitigation |
| `references` | Optional | Standards, advisories, or documentation |
| `fix` | Optional | Short remediation summary |

The engine does not emit dedicated `cwe` or `owasp` fields. Use `references`
when those classifications help the reader.

## 3. Simple Query

```yaml
query:
  from: entry_function
  select: delegatecall
  where:
    - receiver:
        tainted: user_controlled
```

- `from` chooses the units to evaluate. It defaults to `entry_function`.
- `select` chooses the AST kind that anchors the finding.
- `where` is a list of matchers combined with AND. It can mix scope
  preconditions and AST or semantic evidence.

`select` is optional only when the root `where` matcher provides actionable
evidence, such as a sequence, structural `has`, or source regex. A context-only
query cannot produce a useful finding and is rejected.

### Common scopes

| `from` | Evaluates |
|---|---|
| `entry_function` | Public or external state-mutating entry points, including reachable internal code |
| `function` | Every function |
| `main_contract` | Deployable contracts with linearized inherited members |
| `contract` | Contract declarations |
| `library` | Library declarations |
| `abstract` | Abstract declarations |
| `any_contract` | Contracts, interfaces, and libraries |
| `source` | Whole source files for a root regex matcher |

Inheritance, C3 linearization, virtual dispatch, and `super` dispatch are
automatic. There is no `runtime` query option.

## 4. Matchers

Every item under `where` is a matcher map. Common leaf matchers are:

| Matcher | Purpose |
|---|---|
| `block: <kind>` | Match an AST or semantic block kind |
| `name: <regex>` | Match a node, call, or identifier name |
| `regex: <regex>` | Match text within the current source scope |
| `tainted: <source>` | Require provenance from a supported source |
| `func_name: <regex>` | Restrict the enclosing function |
| `modifier: <regex>` | Restrict applied modifier names |
| `visibility: <list>` | Match one of comma-separated visibility values |
| `mutability: <list>` | Match one of comma-separated mutability values |
| `version: <constraint>` | Match the Solidity pragma range |
| `preset: ...` | Apply a named safety or lifecycle property |
| `<attribute>: <regex>` | Match a documented semantic attribute |

Supported taint sources include `parameter`, `state_var`, `local_var`, `sender`,
and `user_controlled`. `user_controlled` includes parameters and caller identity
that the engine can prove.

### Structure and logic

```yaml
- has:
    block: state_write
- in:
    block: unchecked
- arg.0:
    tainted: parameter
- any:
    - block: builtin_send
    - block: lowlevel_call
- not:
    preset: reentrancy_guarded
```

| Matcher | Purpose |
|---|---|
| `has` | Some descendant matches |
| `in` | Some ancestor matches |
| `arg.N` | Positional argument `N` matches |
| `arg.any` | Some positional argument matches |
| `left`, `right` | Direct operand matches |
| `receiver` | The call's direct receiver matches a nested rule |
| `sequence` | Ordered same-execution-path evidence |
| `guarded_by` | Inline guard or exact applied modifier body matches |
| `any` | OR over matcher alternatives |
| `and` | Explicit matcher-level AND |
| `not` | Negate one matcher |
| `unchecked_var:` | Presence-only unchecked arithmetic predicate |

`receiver` takes a nested matcher, not a Boolean:

```yaml
- receiver:
    tainted: user_controlled
```

`unchecked_var` is written as a presence-only marker:

```yaml
- unchecked_var:
```

Do not write `receiver: true` or `unchecked_var: true`. Use nested options or
presence syntax.

## 5. Direct Semantic Presets

Presets are positive properties. Wrap a preset in `not:` to find its absence.

### Protected authorization

```yaml
- preset:
    authorization:
      type: protected
      principal: [owner, pair, pool]
```

`principal` is optional. It may be one string, a non-empty list of OR
alternatives, or an explicit regex option:

```yaml
- preset:
    authorization:
      type: protected
      principal: {regex: '(?i)^(owner|admin|governor)$'}
```

Literal principals are trimmed, deduplicated, sorted, and matched as complete
values. A positive match requires proven dominating authorization through exact
modifier, helper, state-principal, role, and inherited dispatch evidence. A
generic result check such as `require(ok)` is not authorization.

### Caller self-scoping

```yaml
- preset:
    authorization:
      type: self_sender
```

Use `self_sender` when any caller may act only on that caller's own resource. It
does not accept `principal`.

### Initializer first-call takeover

```yaml
- func_name: '(?i)^(initialize|initialise|init|__init|setup)$'
- preset:
    initializer:
      takeover: possible
```

The function regex narrows the intended entries. The preset then requires
positive first-call takeover evidence. Constructor disabling, one-time mutex,
proxy linkage, deployment authority, and atomicity remain internal facts and
are not template attributes.

### Reentrancy guard

```yaml
- not: {preset: reentrancy_guarded}
```

The engine proves a mutex from an applied modifier body when possible. A name is
only a fallback when the body cannot be resolved.

There are no raw authorization or lifecycle attributes, runtime syntax,
candidate pack, compatibility mapping, or authored `unknown` value. Missing or
ambiguous positive evidence fails closed.

## 6. Semantic Attributes

Most detectors use ordinary AST structure. Semantic attributes refine nodes
when the mechanism requires stronger evidence. Important v0.6.0 families are:

### External result handling

```yaml
- any:
    - and:
        - boundary_result_disposition: ^discarded$
        - boundary_result_checked: ^no$
    - and:
        - boundary_result_disposition: ^captured$
        - boundary_result_checked: ^no$
```

This reports discarded and captured-but-unchecked results. Checked, returned,
consumed, unavailable, or ambiguous evidence does not satisfy the detector. An
assignment by itself is not a safety proof.

### Transaction-context accounting

```yaml
- has:
    attr:
      transaction_context_source: ^msg_value$
      transaction_context_use: ^(state_write|call_value|stateful_internal_call)$
      transaction_context_repeatability: ^possible$
```

This requires a proven repeatable transaction-level `msg.value` effect. A read
in an event, condition, local, return, or ordinary external argument is not
enough.

Use the engine reference for the full block-kind and attribute catalogs. Unknown
attribute names fail at load.

## 7. Query Composition

A query may use one-level `or:` or `and:` instead of a simple query.

Use `or:` when alternative shapes should emit the same detector ID:

```yaml
query:
  from: entry_function
  or:
    - select: selfdestruct
      where:
        - not: {preset: {authorization: {type: protected}}}
    - select: delegatecall
      where:
        - receiver: {tainted: user_controlled}
```

Use labeled `and:` branches when several contributing sites must exist in one
shared scope. Each positive branch needs actionable evidence. Query composition
cannot be mixed with top-level `select` or `where` in the same query.

## 8. Complete Example

```yaml
meta:
  id: HIGH-UNPROTECTED-INITIALIZER
  title: Unprotected initializer
  severity: HIGH
  confidence: HIGH
  description: >
    An initializer entry can be claimed by its first caller and writes state.
  recommendation: >
    Apply a one-time initializer guard and restrict deployment initialization.

query:
  from: entry_function
  where:
    - func_name: '(?i)^(initialize|initialise|init|__init|setup)$'
    - preset:
        initializer:
          takeover: possible
    - not:
        preset:
          authorization:
            type: protected
    - has:
        block: state_write
```

The regex identifies initializer intent. The lifecycle preset proves that the
first caller can take over. The authorization preset excludes independently
protected entries. The state-write matcher supplies concrete vulnerability
evidence and an actionable anchor.

## 9. Authoring Checklist

- The ID, filename, folder, and severity agree.
- Metadata says only what the query proves.
- The finding has a precise actionable anchor.
- Semantic evidence is preferred over names and raw source patterns.
- Vulnerable and safe fixtures are present.
- Ambiguous and unresolved evidence fails closed.
- Repeated source and cached-database scans agree.
- `AI.md`, `README.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, and relevant docs are
  updated.
- The downloadable and embedded `official/` trees are byte-identical.
