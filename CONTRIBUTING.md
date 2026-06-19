# Contributing to w3goaudit-templates

Thank you for helping improve the official security rule pack for
[`w3goaudit`](https://github.com/th13vn/w3goaudit). This guide explains how to
write, test, and submit a WQL template.

---

## 1. Anatomy of a template

Every template is a single YAML file with two top-level blocks:

```yaml
meta:
  # human + machine metadata about the finding
query:
  # how the engine decides what to flag
```

A full reference lives in [`docs/template-format.md`](./docs/template-format.md).
Read it before writing a rule.

---

## 2. Required `meta` fields

| Field | Required | Notes |
| --- | --- | --- |
| `id` | yes | Stable, unique. Format `SEC-<CATEGORY>-NNN` (e.g. `SEC-REENTRANCY-002`). Never reuse an id for a different rule. |
| `title` | yes | Short, specific, human-readable. |
| `severity` | yes | One of `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `INFO`. |
| `confidence` | yes | One of `HIGH`, `MEDIUM`, `LOW`. |
| `description` | yes | What the pattern is and why it is dangerous. |
| `recommendation` | yes | Concrete, actionable remediation. |
| `cwe` | recommended | CWE identifier(s) if applicable. |
| `owasp` | recommended | **OWASP Smart Contract Top 10 (2025)** category (e.g. `SC05:2025`). |
| `references` | optional | SWC / best-practice / advisory links. |

Severity and confidence definitions and the OWASP SC Top 10 (2025) mapping are
in [`docs/severity-taxonomy.md`](./docs/severity-taxonomy.md). Classify against
that smart-contract taxonomy — do **not** invent ad-hoc categories.

---

## 3. The `filter` vs `match` layering rule

This is the most important authoring concept. The two layers serve different
purposes and must not be conflated:

- **`filter:`** expresses **function/contract preconditions** — *which*
  functions or contracts are eligible at all. Filters are typically presets:
  `unAuthenticated`, `unLocked` (no reentrancy guard), `entrypoint` scope, etc.
  A filter answers "is this the kind of function where the bug could matter?"

- **`match:`** expresses the **AST pattern** that, if found inside a filtered
  function, constitutes the finding. A match answers "does the dangerous code
  shape actually appear here?"

Keep preconditions in `filter` and code shapes in `match`. Pushing
function-level preconditions into `match` (or vice-versa) makes rules slower,
noisier, and harder to maintain.

```yaml
query:
  scope: entrypoint
  filter:
    preset: unLocked        # precondition: no reentrancy guard
  match:                    # AST pattern: external call before state write
    sequence:
      - any:
          - kind: eth_transfer
          - kind: delegatecall
      - kind: state_write
```

---

## 4. Testing guidance

A rule is not acceptable until it is validated against **both**:

1. A **vulnerable** contract — the rule must fire (true positive).
2. A **safe** contract that is structurally similar — the rule must **not** fire
   (no false positive). The safe variant should include the canonical fix
   (e.g. a reentrancy guard, an access-control modifier, a checked return
   value).

Run the engine locally against your fixtures:

```bash
w3goaudit ./path/to/fixtures/ --template ./official/
# or target a single rule:
w3goaudit ./path/to/fixtures/ --template ./official/your-rule.yaml
```

Prefer tightening a rule to eliminate a false-positive class over broadening it
for one more true positive. Document non-obvious FP/TP trade-offs in a YAML
comment inside the rule (see `reentrancy-pattern.yaml` for an example).

---

## 5. Naming conventions

- **File name:** lowercase, hyphen-separated, descriptive of the pattern, e.g.
  `delegatecall-user-input.yaml`, `unchecked-erc20-transfer.yaml`.
- **`meta.id`:** `SEC-<CATEGORY>-NNN`. Pick a category prefix consistent with
  existing rules (`DELEG`, `REENTRANCY`, `MATH`, `ERC20`, `PRNG`, …). Introduce
  a new prefix only when no existing one fits.
- One vulnerability pattern per file.

---

## 6. PR / review process

1. Fork and create a branch (`add-rule-<id>` or `fix-<id>-fp`).
2. Add or modify the rule under `official/`.
3. Include your vulnerable + safe fixtures (or describe them in the PR) and the
   command output showing the rule fires/does-not-fire as intended.
4. Update the category table in `README.md` and add a `CHANGELOG.md` entry.
5. Open a PR describing the pattern, severity/confidence rationale, and the
   OWASP SC Top 10 (2025) category.
6. A maintainer reviews for correctness, false-positive risk, taxonomy
   accuracy, and naming. Tuning of severity/confidence may be requested before
   merge.

New templates ship in the next `MINOR` release; FP/FN tuning ships in a `PATCH`
release.
