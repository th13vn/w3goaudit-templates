# w3goaudit-templates

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Release](https://img.shields.io/github/v/release/th13vn/w3goaudit-templates?label=release)](https://github.com/th13vn/w3goaudit-templates/releases)
[![Templates](https://img.shields.io/badge/templates-20-blue.svg)](./official)

The official deterministic **WQL** security rule pack for
[`w3goaudit`](https://github.com/th13vn/w3goaudit), a Go static-analysis engine
for Solidity smart contracts.

> Pack `v3.0.0` requires W3GoAudit `v0.6.0` or newer. Upgrade the engine before
> installing this pack because its authorization, initializer, external-result,
> and transaction-context predicates depend on v0.6.0 semantic evidence.

The trusted pack contains **20 evidence-backed detectors**: 2 critical, 10 high,
and 8 medium. Version 3 deliberately removes weakly evidenced and
protocol-specific detectors from the official inventory instead of preserving
them through broad regexes or detector-specific engine shortcuts. Removed
detectors are deferred until generic semantic evidence can prove their
mechanisms.

## Install and Update

Install the engine:

```bash
go install github.com/th13vn/w3goaudit/cmd/w3goaudit@v0.6.0
```

The first scan downloads the latest release of this repository into
`~/.w3goaudit/templates`. Refresh it explicitly at any time:

```bash
w3goaudit --update-templates
```

W3GoAudit writes the installed release tag to
`~/.w3goaudit/templates/.version`. If the network is unavailable, the binary
uses its byte-identical embedded `official/` pack.

Template precedence is:

1. An explicit `--template <file|dir>`.
2. The managed home at `~/.w3goaudit/templates/`.
3. The embedded official pack.

## Repository Layout

```text
w3goaudit-templates/
├── official/
│   ├── critical/     # 2 detectors
│   ├── high/         # 10 detectors
│   └── medium/       # 8 detectors
├── docs/
│   ├── template-format.md
│   └── severity-taxonomy.md
├── AI.md
├── CHANGELOG.md
├── CONTRIBUTING.md
└── README.md
```

## WQL at a Glance

Every template has `meta` plus one `query`. A normal query uses
`select`/`from`/`where`; query-level `and:` and `or:` cover multi-site and
alternative shapes. Inheritance and dispatch are automatic. There is no runtime
syntax or compatibility mapping.

```yaml
meta:
  id: CRITICAL-DELEGATECALL-USER-INPUT
  title: Delegatecall to user-controlled address
  severity: CRITICAL
  confidence: HIGH
  description: >
    A caller-controlled address is used as a delegatecall target.
  recommendation: >
    Use only trusted implementations selected by protected governance.

query:
  from: entry_function
  select: delegatecall
  where:
    - not:
        preset:
          authorization:
            type: protected
    - receiver:
        tainted: user_controlled
```

### Authorization

Use the direct authorization preset. `protected` proves a dominating check for
a trusted principal; `self_sender` proves that callers may act only on their
own resource.

```yaml
- preset:
    authorization:
      type: protected
      principal: [owner, pair, pool]
```

The principal can also be one string or an explicit regex option:

```yaml
- preset:
    authorization:
      type: protected
      principal: {regex: '(?i)^(owner|admin|governor)$'}
```

Use `not:` around the preset to find missing protection. `self_sender` does not
accept a principal.

### Initializer takeover

Select initializer-like entry functions with ordinary WQL, then require the
narrow takeover predicate:

```yaml
- func_name: '(?i)^(initialize|initialise|init|__init|setup)$'
- preset:
    initializer:
      takeover: possible
```

Proxy and constructor details are internal lifecycle evidence, not a public
option list.

### Presence and refinement

Use a nested matcher or a presence-only marker, without `true`:

```yaml
- receiver:
    tainted: parameter
- unchecked_var:
```

Semantic attributes combine with normal regex and structured options. Missing,
unresolved, ambiguous, or incomplete positive evidence fails closed. Templates
do not author `unknown`. Candidate packs, raw lifecycle/auth fields, beginner
and expert dialects, and runtime syntax are not supported surfaces.

Read [Template Format](./docs/template-format.md) for the complete pack-facing
grammar and
[`w3goaudit/docs/wql-syntax.md`](https://github.com/th13vn/w3goaudit/blob/main/docs/wql-syntax.md)
for the authoritative engine reference.

## Official Inventory

| ID | Title | Severity | Confidence |
|---|---|---|---|
| `CRITICAL-DELEGATECALL-USER-INPUT` | Delegatecall to user-controlled address | CRITICAL | HIGH |
| `CRITICAL-SELFDESTRUCT-UNPROTECTED` | Unprotected selfdestruct | CRITICAL | HIGH |
| `HIGH-ARBITRARY-LOW-LEVEL-CALL` | Arbitrary low-level call (user-controlled target and calldata) | HIGH | MEDIUM |
| `HIGH-ENCODE-PACKED-COLLISION` | Hash collision via abi.encodePacked with multiple dynamic arguments | HIGH | MEDIUM |
| `HIGH-INCORRECT-EXP` | Suspicious use of ^ (bitwise XOR, likely meant **) | HIGH | MEDIUM |
| `HIGH-MSG-VALUE-IN-LOOP` | msg.value effect repeats inside a loop | HIGH | MEDIUM |
| `HIGH-NO-SLIPPAGE-CHECK` | AMM swap with no slippage bound (amountOutMin = 0) | HIGH | MEDIUM |
| `HIGH-REENTRANCY-PATTERN` | Potential reentrancy: state variable modification | HIGH | MEDIUM |
| `HIGH-UNCHECKED-ERC20-TRANSFER` | Unchecked ERC20 transfer or transferFrom return value | HIGH | MEDIUM |
| `HIGH-UNPROTECTED-INITIALIZER` | Unprotected initializer (anyone can become owner) | HIGH | HIGH |
| `HIGH-UNRESTRICTED-TRANSFEROWNERSHIP` | Unrestricted transferOwnership | HIGH | MEDIUM |
| `HIGH-WEAK-PRNG` | Weak randomness from block variables | HIGH | MEDIUM |
| `MEDIUM-BIDI-CHARACTERS` | Unicode bidirectional control characters in source (Trojan Source) | MEDIUM | HIGH |
| `MEDIUM-DANGEROUS-STRICT-EQUALITY` | Dangerous strict equality on externally influenced balance | MEDIUM | HIGH |
| `MEDIUM-DIVIDE-BEFORE-MULTIPLY` | Division before multiplication (precision loss) | MEDIUM | HIGH |
| `MEDIUM-TX-ORIGIN-AUTH` | tx.origin used for authentication | MEDIUM | HIGH |
| `MEDIUM-UNCHECKED-ARITHMETIC` | Arithmetic in unchecked block (potential overflow) | MEDIUM | MEDIUM |
| `MEDIUM-UNCHECKED-LOWLEVEL-CALL` | Unchecked low-level call return value | MEDIUM | MEDIUM |
| `MEDIUM-UNCHECKED-SEND` | Unchecked send() return value | MEDIUM | HIGH |
| `MEDIUM-VIEW-PURE-MODIFIES-STATE` | view/pure function modifies state | MEDIUM | HIGH |

## Evidence-Backed Families

- Unchecked ERC20, low-level-call, and `send` detectors use semantic result
  disposition and direct-check evidence. Assigning a result is not treated as a
  safety proof.
- `HIGH-MSG-VALUE-IN-LOOP` requires a proven repeatable state write, ETH call
  value, or exact stateful helper. Merely reading `msg.value` does not report.
- Protected-call detectors use exact authorization evidence through modifiers,
  helpers, state principals, roles, and dispatched inheritance.
- `HIGH-UNPROTECTED-INITIALIZER` requires positive first-call takeover evidence
  plus an initializer-shaped entry and state write.

## Versioning

- **MAJOR**: remove or rename detector IDs, or require a breaking authoring
  migration.
- **MINOR**: add detectors without removing IDs.
- **PATCH**: tune precision, metadata, or documentation without an inventory
  break.

Each GitHub Release tag is the downloadable artifact. No separate binary or
archive upload is required. Before release, this repository's `official/` tree
must be byte-identical to the embedded pack in W3GoAudit.

## Contributing

Read [CONTRIBUTING.md](./CONTRIBUTING.md). Every detector needs vulnerable and
safe fixtures, evidence-level justification, source/cache parity, and validation
through the W3GoAudit engine.

## License

[MIT](./LICENSE)
