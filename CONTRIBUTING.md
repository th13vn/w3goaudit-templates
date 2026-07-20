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
| `id` | yes | Stable, unique. Format `<SEVERITY>-<FILENAME>` — the upper-cased file name (e.g. `HIGH-REENTRANCY-BALANCE` for `high/reentrancy-balance.yaml`). Never reuse an id for a different rule. |
| `title` | yes | Short, specific, human-readable. |
| `severity` | yes | One of `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `INFO`. |
| `confidence` | yes | One of `HIGH`, `MEDIUM`, `LOW`. |
| `description` | recommended | What the pattern is and why it is dangerous. |
| `recommendation` | recommended | Concrete, actionable remediation. |
| `references` | optional | SWC / best-practice / advisory links (engine propagates these to findings). |
| `fix` | optional | Short one-line fix summary, surfaced alongside `recommendation`. |

> **No `cwe` / `owasp` fields:** the engine does not parse or emit dedicated
> `cwe`/`owasp` meta fields — they are silently ignored. Record any CWE / OWASP
> SC Top 10 classification as a `references` entry instead.

Severity and confidence definitions and the (advisory) OWASP SC Top 10 (2025)
mapping are in [`docs/severity-taxonomy.md`](./docs/severity-taxonomy.md).
Classify against that smart-contract taxonomy — do **not** invent ad-hoc
categories.

---

## 3. The `select` / `from` / `where` model

A query has three parts. `where` is one flat matcher list that mixes
**preconditions** and **code shapes** — the engine routes each matcher to the
right layer, so you do not split them into separate blocks:

- **`from:`** — the scope to iterate (`entry_function` default, or `function`,
  `contract`, `main_contract`, `source`, …). Answers "which units does this rule
  even look at?"

- **`select:`** — the block kind whose matched node anchors the finding
  (optional when a top-level `sequence:` / `has:` / root matcher already
  anchors). Answers "what is the finding pinned to?"

- **`where:`** — the matchers. Preconditions read as presets and
  function/contract predicates (`func_name`, `modifier`, `version`, …); code
  shapes read as `block:` / `has:` / `sequence:` / `arg.N:` / `tainted:`. Express
  a safety property's **absence** with `not: { preset: ... }` — each preset is
  true when its property is present (`access_controlled`, `caller_checked`,
  `reentrancy_guarded`).

For "shape A **or** shape B" use a top-level `or:` list of branches; for a
multi-site combination finding use a top-level `and:` of labeled branches.

```yaml
query:
  from: entry_function
  where:
    - not: { preset: reentrancy_guarded }   # precondition: no reentrancy guard
    - sequence:                             # AST shape: external call before state write
        - any:
            - block: eth_transfer
            - block: delegatecall
        - block: state_write
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
w3goaudit ./path/to/fixtures/ --template ./official/high/your-rule.yaml
```

Prefer tightening a rule to eliminate a false-positive class over broadening it
for one more true positive. Document non-obvious FP/TP trade-offs in a YAML
comment inside the rule (see `reentrancy-pattern.yaml` for an example).

---

## 5. Naming conventions

- **File name:** lowercase, hyphen-separated, descriptive of the pattern, e.g.
  `delegatecall-user-input.yaml`, `unchecked-erc20-transfer.yaml`.
- **Folder:** place the file under `official/<severity>/` matching its
  `meta.severity` — `critical/`, `high/`, or `medium/`.
- **`meta.id`:** `<SEVERITY>-<FILENAME>` — the severity followed by the
  upper-cased file name (hyphens kept), e.g. `high/delegatecall-in-loop.yaml`
  → `HIGH-DELEGATECALL-IN-LOOP`. The id must stay in sync with both the file
  name and the folder.
- One vulnerability pattern per file.

---

## 6. PR / review process

1. Fork and create a branch (`add-rule-<id>` or `fix-<id>-fp`).
2. Add or modify the rule under `official/`.
3. Include your vulnerable + safe fixtures (or describe them in the PR) and the
   command output showing the rule fires/does-not-fire as intended.
4. Update the category table in `README.md` and add a `CHANGELOG.md` entry.
5. Open a PR describing the pattern and the severity/confidence rationale
   (noting the OWASP SC Top 10 (2025) category for context, even though it is
   not an emitted field).
6. A maintainer reviews for correctness, false-positive risk, taxonomy
   accuracy, and naming. Tuning of severity/confidence may be requested before
   merge.

New templates ship in the next `MINOR` release; FP/FN tuning ships in a `PATCH`
release.
