# w3goaudit-templates Index

AI-oriented map of the official WQL template pack. This repository is the
canonical, versioned home of the rules the [`w3goaudit`](https://github.com/th13vn/w3goaudit)
engine downloads into the user's template home.

## Purpose

Each file under `official/` is one YAML detector written in canonical **WQL**
(`select` / `from` / `where`, or an `and:` / `or:` composition). The engine
loads every `*.yaml` recursively, evaluates each rule against the parsed
Solidity AST, and emits a finding per match. Distributing rules here (separate
from the engine) lets coverage ship via tagged GitHub Releases without
rebuilding the tool.

## Repository Map

| Path | Responsibility |
|---|---|
| `official/critical/` | CRITICAL-severity detectors (2) |
| `official/high/` | HIGH-severity detectors (15) |
| `official/medium/` | MEDIUM-severity detectors (8) |
| `docs/template-format.md` | WQL template-format reference (meta + query grammar) |
| `docs/severity-taxonomy.md` | Severity/confidence definitions + advisory OWASP SC Top 10 (2025) mapping |
| `README.md` | User-facing overview, consumption model, template catalog |
| `CONTRIBUTING.md` | Authoring rules, required `meta` fields, testing guidance |
| `CHANGELOG.md` | Versioned release history |
| `LICENSE` | MIT |

**25 templates total** (2 critical + 15 high + 8 medium).

## Template Catalog

| Severity | ID | File |
|---|---|---|
| CRITICAL | `CRITICAL-DELEGATECALL-USER-INPUT` | `official/critical/delegatecall-user-input.yaml` |
| CRITICAL | `CRITICAL-SELFDESTRUCT-UNPROTECTED` | `official/critical/selfdestruct-unprotected.yaml` |
| HIGH | `HIGH-ARBITRARY-LOW-LEVEL-CALL` | `official/high/arbitrary-low-level-call.yaml` |
| HIGH | `HIGH-ARBITRARY-SEND-ETH` | `official/high/arbitrary-send-eth.yaml` |
| HIGH | `HIGH-ARBITRARY-TRANSFERFROM` | `official/high/arbitrary-transferfrom.yaml` |
| HIGH | `HIGH-DELEGATECALL-IN-LOOP` | `official/high/delegatecall-in-loop.yaml` |
| HIGH | `HIGH-ECDSA-RECOVER-MALLEABLE` | `official/high/ecdsa-recover-malleable.yaml` |
| HIGH | `HIGH-ENCODE-PACKED-COLLISION` | `official/high/encode-packed-collision.yaml` |
| HIGH | `HIGH-INCORRECT-EXP` | `official/high/incorrect-exp.yaml` |
| HIGH | `HIGH-MSG-VALUE-IN-LOOP` | `official/high/msg-value-in-loop.yaml` |
| HIGH | `HIGH-PROXY-STORAGE-COLLISION` | `official/high/proxy-storage-collision.yaml` |
| HIGH | `HIGH-REENTRANCY-BALANCE` | `official/high/reentrancy-balance.yaml` |
| HIGH | `HIGH-REENTRANCY-PATTERN` | `official/high/reentrancy-pattern.yaml` |
| HIGH | `HIGH-UNCHECKED-ERC20-TRANSFER` | `official/high/unchecked-erc20-transfer.yaml` |
| HIGH | `HIGH-UNPROTECTED-INITIALIZER` | `official/high/unprotected-initializer.yaml` |
| HIGH | `HIGH-UNRESTRICTED-TRANSFEROWNERSHIP` | `official/high/unrestricted-transferownership.yaml` |
| HIGH | `HIGH-WEAK-PRNG` | `official/high/weak-prng.yaml` |
| MEDIUM | `MEDIUM-BOOLEAN-CST` | `official/medium/boolean-cst.yaml` |
| MEDIUM | `MEDIUM-DANGEROUS-STRICT-EQUALITY` | `official/medium/dangerous-strict-equality.yaml` |
| MEDIUM | `MEDIUM-DIVIDE-BEFORE-MULTIPLY` | `official/medium/divide-before-multiply.yaml` |
| MEDIUM | `MEDIUM-TX-ORIGIN-AUTH` | `official/medium/tx-origin-auth.yaml` |
| MEDIUM | `MEDIUM-UNCHECKED-ARITHMETIC` | `official/medium/unchecked-arithmetic.yaml` |
| MEDIUM | `MEDIUM-UNCHECKED-LOWLEVEL-CALL` | `official/medium/unchecked-lowlevel-call.yaml` |
| MEDIUM | `MEDIUM-UNCHECKED-SEND` | `official/medium/unchecked-send.yaml` |
| MEDIUM | `MEDIUM-VIEW-PURE-MODIFIES-STATE` | `official/medium/view-pure-modifies-state.yaml` |

## Key Invariants

- **ID form is `<SEVERITY>-<FILENAME>`** (upper-cased file name). The ID appears
  in every finding and in `--include`/`--exclude` globs; never reuse an ID.
- **File lives in `official/<severity>/` matching its `severity` meta.** The
  engine loads recursively, so folder layout is organizational only.
- **Canonical WQL only.** Templates use `query:` with `select`/`from`/`where`
  (or `and:`/`or:`). The pre-2.0 `scope`/`filter`/`match` form is not accepted
  by w3goaudit v0.4+. Requires engine **v0.4.0+**.
- **Presets express presence of a safety property**; the vulnerable case negates
  them (`not: {preset: access_controlled|caller_checked|reentrancy_guarded}`).
- **`meta` fields the engine parses:** `id`, `title`, `severity`, `confidence`,
  `description`, `recommendation`, `references`, `fix`. There are no `cwe`/`owasp`
  fields — record classification under `references`.
- **Every rule must pass a vulnerable + safe fixture pair** before merge
  (CONTRIBUTING §4).

## Change Checklist

- Adding/tuning a rule: place under `official/<severity>/`, set a matching
  `<SEVERITY>-<FILENAME>` id, validate against vulnerable **and** safe fixtures,
  and update the catalog tables in this file, `README.md`, and `CHANGELOG.md`.
- Grammar/behavior questions: the authoritative engine reference is
  [`w3goaudit/docs/wql-syntax.md`](https://github.com/th13vn/w3goaudit/blob/main/docs/wql-syntax.md);
  `docs/template-format.md` here is the pack-facing subset.
- Releases are tagged `vX.Y.Z` (SemVer). MAJOR = removing/renaming templates or
  breaking the WQL schema; MINOR = new templates; PATCH = doc/tuning with
  identical detections.
