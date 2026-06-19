# w3goaudit-templates

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Release](https://img.shields.io/github/v/release/th13vn/w3goaudit-templates?label=release)](https://github.com/th13vn/w3goaudit-templates/releases)
[![Templates](https://img.shields.io/badge/templates-25-blue.svg)](./official)

Official **WQL** (w3goaudit Query Language) security templates for
[`w3goaudit`](https://github.com/th13vn/w3goaudit) — a Go static-analysis engine
for Solidity smart contracts.

This repository is the canonical, versioned home of the official template pack.
Each template is a YAML rule that describes a vulnerability pattern in terms of
function/contract preconditions (`filter`) and an AST match (`match`). The
`w3goaudit` binary downloads this repo's latest GitHub Release and extracts it
into the user's template home so the engine always runs an up-to-date rule set.

---

## Relationship to w3goaudit

`w3goaudit` ships with a small set of templates embedded in the binary, but the
authoritative, regularly-updated rule pack lives **here**. Distributing rules in
a separate, release-tagged repository (nuclei-style) means security coverage can
be improved and shipped to users without rebuilding or re-releasing the tool
itself.

| Component | Repository | Role |
| --- | --- | --- |
| Engine / CLI | [`th13vn/w3goaudit`](https://github.com/th13vn/w3goaudit) | Parses Solidity, evaluates WQL, reports findings |
| Template pack | `th13vn/w3goaudit-templates` (this repo) | The security rules the engine evaluates |

---

## How the tool consumes this repo

On first run, `w3goaudit` resolves the latest tagged release of this repository,
downloads its source zipball, and unzips the tracked files into the template
home:

```
~/.w3goaudit/templates/
├── official/            # the rule pack from this repo
│   ├── reentrancy-pattern.yaml
│   ├── delegatecall-user-input.yaml
│   └── ...
└── .version             # the release tag currently installed (e.g. v1.0.0)
```

The extracted tree mirrors this repository exactly — the `official/` directory
of rules becomes `~/.w3goaudit/templates/official/`.

Refresh the local pack at any time:

```bash
w3goaudit --update-templates
```

This re-resolves the latest release, re-extracts it, and rewrites
`~/.w3goaudit/templates/.version` with the new tag. If the local `.version`
already matches the latest release, the update is a no-op.

---

## Template directory structure

```
w3goaudit-templates/
├── official/            # production rule pack (downloaded to the template home)
│   └── *.yaml
├── docs/                # WQL authoring reference
│   ├── template-format.md
│   └── severity-taxonomy.md
├── README.md
├── CONTRIBUTING.md
├── CHANGELOG.md
└── LICENSE
```

---

## What a WQL template looks like

```yaml
meta:
  id: SEC-DELEG-001
  title: Delegatecall to User-Controlled Address
  severity: CRITICAL
  confidence: HIGH
  description: >
    A delegatecall is executed with a target address that comes from a function
    parameter (user-controlled). An attacker can pass a malicious contract
    address, executing arbitrary code in this contract's context.
  recommendation: >
    Never delegatecall to user-supplied addresses. Maintain a whitelist of
    trusted implementations, or use a timelocked admin role to change them.
  references:
    - https://swcregistry.io/docs/SWC-112

query:
  scope: entrypoint

  filter:
    preset: unAuthenticated      # function/contract precondition

  match:                          # AST pattern that must be present
    contains:
      kind: delegatecall
      contains:
        kind: expr.identifier
        attr:
          call_receiver: true
        tainted_from: parameter
```

The two-layer model is central:

- **`filter:`** narrows *which functions/contracts* are even considered (e.g.
  "unauthenticated entrypoints", "missing reentrancy guard").
- **`match:`** is the AST pattern that, if found inside a filtered function,
  produces a finding.

See [`docs/template-format.md`](./docs/template-format.md) for the full anatomy.

---

## Severity & confidence taxonomy

| Severity | Meaning |
| --- | --- |
| `CRITICAL` | Direct loss of funds / full contract compromise; high exploitability |
| `HIGH` | Serious impact (fund loss, privilege escalation) under realistic conditions |
| `MEDIUM` | Meaningful risk, conditional impact, or strong code-quality/security smell |
| `LOW` | Minor risk or hardening opportunity |
| `INFO` | Informational; no direct security impact |

| Confidence | Meaning |
| --- | --- |
| `HIGH` | Pattern is precise; few false positives expected |
| `MEDIUM` | Heuristic pattern; manual confirmation recommended |
| `LOW` | Broad pattern; expect to triage false positives |

Vulnerability classification follows the **OWASP Smart Contract Top 10 (2025)**
taxonomy. See [`docs/severity-taxonomy.md`](./docs/severity-taxonomy.md) for the
full mapping.

---

## Template categories

The current pack contains **25** official templates:

| ID | Title | Severity | Confidence |
| --- | --- | --- | --- |
| SEC-DELEG-001 | Delegatecall to User-Controlled Address | CRITICAL | HIGH |
| SEC-DEST-001 | Unprotected selfdestruct | CRITICAL | HIGH |
| SEC-CALL-001 | Arbitrary Low-Level Call (User-Controlled Target & Calldata) | HIGH | MEDIUM |
| SEC-ETH-001 | Unprotected ETH Withdrawal | HIGH | MEDIUM |
| SEC-ERC20-001 | Arbitrary transferFrom Call | HIGH | MEDIUM |
| SEC-DELEG-002 | Delegatecall Inside a Loop | HIGH | MEDIUM |
| SEC-SIG-001 | Signature Malleability: Raw Signature Bytes as State Key | HIGH | MEDIUM |
| SEC-HASH-001 | Hash Collision via abi.encodePacked (Multiple Dynamic Args) | HIGH | MEDIUM |
| SEC-MATH-003 | Suspicious Use of `^` (Bitwise XOR, Likely Meant `**`) | HIGH | MEDIUM |
| SEC-MSGVAL-001 | msg.value Used Inside a Loop | HIGH | MEDIUM |
| SEC-PROXY-001 | Proxy Storage Layout Collision | HIGH | MEDIUM |
| SEC-REENTRANCY-002 | Reentrancy via balanceOf Delta (Stale Snapshot) | HIGH | MEDIUM |
| SEC-GEN-REENTRANCY | Potential Reentrancy — State Variable Modification | HIGH | MEDIUM |
| SEC-ERC20-002 | Unchecked ERC20 transfer / transferFrom Return Value | HIGH | MEDIUM |
| SEC-UPGRADE-001 | Unprotected Initializer (Anyone Can Become Owner) | HIGH | HIGH |
| SEC-OWNER-001 | Unrestricted transferOwnership | HIGH | MEDIUM |
| SEC-PRNG-001 | Weak Randomness from Block Variables | HIGH | MEDIUM |
| SEC-BOOL-001 | Boolean Constant Misuse | MEDIUM | MEDIUM |
| SEC-EQ-001 | Dangerous Strict Equality on Externally-Influenced Balance | MEDIUM | HIGH |
| SEC-MATH-002 | Division Before Multiplication (Precision Loss) | MEDIUM | HIGH |
| SEC-TXORIGIN-001 | tx.origin Used for Authentication | MEDIUM | HIGH |
| SEC-MATH-001 | Arithmetic in Unchecked Block (Potential Overflow) | MEDIUM | HIGH |
| SEC-LOWLEVEL-CALL-001 | Unchecked Low-Level Call Return Value | MEDIUM | MEDIUM |
| SEC-SEND-001 | Unchecked send() Return Value | MEDIUM | HIGH |
| SEC-MUTABILITY-001 | view/pure Function Modifies State | MEDIUM | HIGH |

---

## Versioning policy

- Releases are tagged **`vX.Y.Z`** (semantic versioning).
- The `w3goaudit` tool tracks the installed tag in
  `~/.w3goaudit/templates/.version`.
- A release's auto-generated source zipball is the artifact the tool downloads —
  no manual asset upload is required.
- **MAJOR** — removing/renaming templates or breaking the WQL schema in a way
  that requires a newer engine.
- **MINOR** — adding new templates or non-breaking metadata.
- **PATCH** — false-positive/false-negative tuning, doc fixes, wording.

---

## Contributing

New rules and tuning are welcome. Read [`CONTRIBUTING.md`](./CONTRIBUTING.md) for
the required `meta` fields, the `filter`-vs-`match` layering rule, testing
guidance (every rule must be validated against both a vulnerable and a safe
contract), naming conventions, and the review process.

## License

[MIT](./LICENSE) — same license as the `w3goaudit` tool.
