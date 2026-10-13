# w3goaudit-templates Index

Agent-facing map for the official WQL detector pack consumed by
[`w3goaudit`](https://github.com/th13vn/w3goaudit). This repository publishes
the downloadable pack; `w3goaudit/templates/official` is the byte-identical
embedded fallback paired with the same release.

## Purpose

Every file under `official/` is one deterministic Solidity detector written in
canonical WQL. The trusted pack favors mechanisms supported by exact semantic
or structural evidence. Unsupported protocol-specific or weakly evidenced
detectors are deferred rather than approximated with broad names or source
regexes.

Pack `v3.0.1` requires W3GoAudit `v0.7.0` or newer. The current inventory is
**20 detectors**: 2 critical, 10 high, and 8 medium.

## Repository Map

| Path | Responsibility |
|---|---|
| `official/critical/` | CRITICAL detectors (2) |
| `official/high/` | HIGH detectors (10) |
| `official/medium/` | MEDIUM detectors (8) |
| `docs/template-format.md` | Accessible WQL authoring reference |
| `docs/severity-taxonomy.md` | Severity and confidence guidance |
| `README.md` | User-facing installation, model, and inventory |
| `CONTRIBUTING.md` | Rule-quality and submission requirements |
| `CHANGELOG.md` | Versioned release history and breaking changes |

## Canonical Authoring Model

A template contains `meta` plus one `query`. A query uses
`select`/`from`/`where`, or a query-level `and:`/`or:` composition. Inheritance,
C3 linearization, virtual dispatch, and `super` dispatch are automatic engine
behavior. Templates never select a runtime mode.

Authorization is one direct parameterized preset:

```yaml
- not:
    preset:
      authorization:
        type: protected
        principal: [owner, pair, pool]
```

`authorization.type` is `protected` or `self_sender`. `protected` may omit the
principal, name one principal, list OR alternatives, or use the explicit
`principal: {regex: "..."}` option. Literal principals are trimmed,
deduplicated, sorted, and matched as complete values. `self_sender` means a
caller is restricted to its own resource and does not accept `principal`.

Initializer takeover is one narrow preset, normally combined with a function
name regex:

```yaml
- func_name: '(?i)^(initialize|initialise|init|__init|setup)$'
- preset:
    initializer:
      takeover: possible
```

Constructor disabling, mutex state, proxy linkage, deployment authority, and
atomicity remain internal lifecycle evidence. They are not public template
options. Missing, unresolved, ambiguous, or incomplete positive evidence fails
closed.

Use nested matchers and presence syntax instead of Boolean flags:

```yaml
- receiver:
    tainted: user_controlled
- unchecked_var:
```

Do not author `runtime:`, raw authorization or lifecycle attributes, candidate
templates, `unknown` values, Boolean marker values such as
`unchecked_var: true`, or compatibility aliases. Use semantic attributes plus
ordinary regex or structured options when a detector needs refinement.

## Official Detector Inventory

| Severity | ID | File |
|---|---|---|
| CRITICAL | `CRITICAL-DELEGATECALL-USER-INPUT` | `official/critical/delegatecall-user-input.yaml` |
| CRITICAL | `CRITICAL-SELFDESTRUCT-UNPROTECTED` | `official/critical/selfdestruct-unprotected.yaml` |
| HIGH | `HIGH-ARBITRARY-LOW-LEVEL-CALL` | `official/high/arbitrary-low-level-call.yaml` |
| HIGH | `HIGH-ENCODE-PACKED-COLLISION` | `official/high/encode-packed-collision.yaml` |
| HIGH | `HIGH-INCORRECT-EXP` | `official/high/incorrect-exp.yaml` |
| HIGH | `HIGH-MSG-VALUE-IN-LOOP` | `official/high/msg-value-in-loop.yaml` |
| HIGH | `HIGH-NO-SLIPPAGE-CHECK` | `official/high/no-slippage-check.yaml` |
| HIGH | `HIGH-REENTRANCY-PATTERN` | `official/high/reentrancy-pattern.yaml` |
| HIGH | `HIGH-UNCHECKED-ERC20-TRANSFER` | `official/high/unchecked-erc20-transfer.yaml` |
| HIGH | `HIGH-UNPROTECTED-INITIALIZER` | `official/high/unprotected-initializer.yaml` |
| HIGH | `HIGH-UNRESTRICTED-TRANSFEROWNERSHIP` | `official/high/unrestricted-transferownership.yaml` |
| HIGH | `HIGH-WEAK-PRNG` | `official/high/weak-prng.yaml` |
| MEDIUM | `MEDIUM-BIDI-CHARACTERS` | `official/medium/bidi-characters.yaml` |
| MEDIUM | `MEDIUM-DANGEROUS-STRICT-EQUALITY` | `official/medium/dangerous-strict-equality.yaml` |
| MEDIUM | `MEDIUM-DIVIDE-BEFORE-MULTIPLY` | `official/medium/divide-before-multiply.yaml` |
| MEDIUM | `MEDIUM-TX-ORIGIN-AUTH` | `official/medium/tx-origin-auth.yaml` |
| MEDIUM | `MEDIUM-UNCHECKED-ARITHMETIC` | `official/medium/unchecked-arithmetic.yaml` |
| MEDIUM | `MEDIUM-UNCHECKED-LOWLEVEL-CALL` | `official/medium/unchecked-lowlevel-call.yaml` |
| MEDIUM | `MEDIUM-UNCHECKED-SEND` | `official/medium/unchecked-send.yaml` |
| MEDIUM | `MEDIUM-VIEW-PURE-MODIFIES-STATE` | `official/medium/view-pure-modifies-state.yaml` |

## Evidence-Sensitive Detectors

- `HIGH-UNCHECKED-ERC20-TRANSFER`, `MEDIUM-UNCHECKED-LOWLEVEL-CALL`, and
  `MEDIUM-UNCHECKED-SEND` use external-boundary result disposition and direct
  check evidence. Discarded and captured-but-unchecked results report; checked,
  returned, consumed, unavailable, or ambiguous results do not.
- `HIGH-MSG-VALUE-IN-LOOP` uses transaction-context facts. A loop must repeat a
  proven `msg.value` state write, ETH call value, or exact stateful helper. A
  textual read alone is not enough.
- Protected-entry detectors use `preset.authorization`; initializer takeover
  uses `preset.initializer`. Generic `require(ok)` checks are not authorization.

## Release and Sync Invariants

- Release tags use `vX.Y.Z`. Removing or renaming detector IDs requires a major
  release.
- Pack `v3.0.1` is paired with core `v0.7.0` and must be published only after
  that engine release exists. It uses intentional_divide_rounding, a new matcher
  unavailable in v0.6.0.
- Before release, `official/` here and `w3goaudit/templates/official/` must be
  byte-identical. Never publish only one distribution channel.
- The W3GoAudit engine is the validator. Run official-pack tests, source/cache
  parity, the competitive benchmark, and a real scan with this directory.
- A GitHub Release tag is the downloadable artifact. W3GoAudit records the tag
  in `~/.w3goaudit/templates/.version`.

## Change Checklist

1. Keep the file path, severity, and `<SEVERITY>-<FILENAME>` ID aligned.
2. Add vulnerable, safe, and structural-variant fixtures in the core repo.
3. Prefer semantic, type, taint, reachability, and ordering evidence over names.
4. Validate malformed and ambiguous cases fail closed.
5. Compare repeated source scans and a cached-database scan.
6. Update this file, `README.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, and `docs/`
   whenever inventory or authoring behavior changes.
7. Synchronize both pack copies and release the engine first when new semantics
   are required.

The complete engine grammar is maintained in
[`w3goaudit/docs/wql-syntax.md`](https://github.com/th13vn/w3goaudit/blob/main/docs/wql-syntax.md).
