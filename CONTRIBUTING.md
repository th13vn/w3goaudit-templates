# Contributing to w3goaudit-templates

Thank you for improving the official security rule pack for
[`w3goaudit`](https://github.com/th13vn/w3goaudit). The goal is not to maximize
the number of warnings. The goal is to encode vulnerability mechanisms with
evidence that an auditor can verify quickly.

Pack `v3.0.1` requires W3GoAudit `v0.7.0` or newer. Use the paired W3GoAudit
`v0.7.0` engine when validating a rule.

## 1. Template Shape

Every template is one YAML file with two top-level blocks:

```yaml
meta:
  # finding metadata
query:
  # canonical WQL
```

A query uses `select`/`from`/`where`, or a one-level query composition with
`and:` or `or:`. There is one authoring language, not separate beginner, expert,
legacy, or runtime layers. Read [Template Format](./docs/template-format.md)
before adding a rule.

## 2. Required Metadata

Official detectors require all fields below.

| Field | Requirement |
|---|---|
| `id` | Stable `<SEVERITY>-<FILENAME>` value that matches path and severity |
| `title` | Concise sentence-case mechanism summary |
| `severity` | `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, or `INFO` |
| `confidence` | `HIGH`, `MEDIUM`, or `LOW` |
| `description` | State exactly what the query proves and why it matters |
| `recommendation` | Practical mitigation for the proven condition |

`references` and `fix` are optional. The engine does not emit dedicated `cwe`
or `owasp` metadata. Put such classifications in `references` if useful.

Comments should explain a non-obvious matcher boundary or semantic scope. Do
not keep implementation history, benchmark anecdotes, or migration notes in a
production detector file.

## 3. Authoring Principles

### Prefer evidence to vocabulary

Use semantic type, taint, authorization, reachability, result, transaction
context, and execution-order evidence before names or source regex. A name
regex is appropriate for narrowing intent, such as initializer entry names, but
is not a substitute for the vulnerable mechanism.

### Use direct semantic presets

Protected principal:

```yaml
- not:
    preset:
      authorization:
        type: protected
        principal: [owner, pair, pool]
```

Caller self-scoping:

```yaml
- preset:
    authorization:
      type: self_sender
```

Initializer takeover:

```yaml
- func_name: '(?i)^(initialize|initialise|init|__init|setup)$'
- preset:
    initializer:
      takeover: possible
```

`protected` may omit `principal`, use one literal, use a list of OR literals, or
use `principal: {regex: "..."}`. `self_sender` does not accept a principal.

### Keep syntax explicit and small

- Inheritance, virtual dispatch, and `super` dispatch are automatic. Do not add
  `runtime:` syntax.
- Do not author internal authorization or initializer fact attributes.
- Do not create candidate templates inside the official pack.
- Do not author `unknown`; missing positive evidence fails closed.
- Do not write Boolean presence flags. Use `unchecked_var:` and nested options
  such as `receiver: {tainted: parameter}`.
- Do not introduce compatibility aliases or mapping layers.
- Use ordinary attributes, regex, and structured options for refinement.

## 4. Evidence and Test Requirements

A rule is not ready until the core repository includes all applicable cases:

1. A vulnerable fixture that must report.
2. A safe fixture with the canonical mitigation that must remain silent.
3. Structural variants such as internal helpers, inherited functions, overloads,
   modifiers, or Yul when relevant.
4. Ambiguous, unresolved, or incomplete controls that must fail closed.
5. Repeated execution and source/cache parity.

Run the engine against the pack:

```bash
go build -trimpath -o /tmp/w3goaudit ./cmd/w3goaudit
/tmp/w3goaudit test-data/security \
  --template ../w3goaudit-templates/official \
  --output /tmp/w3goaudit-pack-scan \
  --no-color
```

Run focused and full engine tests from the W3GoAudit repository:

```bash
go test ./pkg/engine -run 'TestOfficialPack|TestPromotedDetectors' -count=1
go test ./... -count=1
```

Before release, compare both distribution copies:

```bash
diff -ru templates/official ../w3goaudit-templates/official
```

## 5. Naming and Layout

- Use lowercase hyphen-separated filenames.
- Place the file under `official/<severity>/`.
- Set `meta.id` to the uppercased severity plus filename, for example
  `high/example-rule.yaml` becomes `HIGH-EXAMPLE-RULE`.
- Keep one vulnerability mechanism per detector ID.
- Never reuse a removed ID for a different mechanism.

## 6. Review Standard

A reviewer checks:

- the rule proves what its description claims;
- the finding anchors on an actionable source location;
- safe patterns and ambiguous evidence do not report;
- the matcher is protocol-neutral unless the detector explicitly and reliably
  models a protocol standard;
- exact identities and inherited dispatch remain correct;
- the pack is deterministic and byte-identical across both repositories;
- inventory tables and active docs are updated.

If the engine lacks necessary evidence, defer the detector and propose a generic
semantic capability. Do not compensate with a broad production regex.

## 7. Versioning and Pull Requests

- Removing or renaming IDs is a major release.
- Adding IDs is a minor release.
- Precision tuning and documentation are patch releases when inventory remains
  compatible.

A pull request must list the vulnerable and safe cases, explain the evidence and
false-positive boundary, report the exact verification commands, update
`README.md`, `AI.md`, `CHANGELOG.md`, and relevant docs, and identify the minimum
W3GoAudit version.
