# Severity, Confidence & OWASP SC Top 10 (2025) Taxonomy

Every official template classifies its finding along three axes: **severity**
(impact), **confidence** (precision of the rule), and an **OWASP Smart Contract
Top 10 (2025)** category. This document is the authoritative reference for all
three.

---

## 1. Severity levels

Severity describes the **impact and exploitability** of the issue if the matched
pattern is a true positive.

| Level | Definition | Typical examples |
| --- | --- | --- |
| `CRITICAL` | Direct, near-certain loss of funds or total contract compromise; little or no precondition. | Delegatecall to a user-controlled address, unprotected `selfdestruct`. |
| `HIGH` | Serious impact — fund loss, privilege escalation, or unrecoverable corruption — under realistic conditions. | Unprotected ETH withdrawal, reentrancy, unprotected initializer, arbitrary `transferFrom`, weak PRNG used for value. |
| `MEDIUM` | Meaningful risk with a conditional impact, or a strong security/correctness smell that frequently precedes a bug. | Unchecked low-level/`send` return value, `tx.origin` auth, division-before-multiply, strict equality on balances. |
| `LOW` | Minor risk or a hardening opportunity; limited or indirect impact. | Defensive-coding gaps, low-impact gas/footgun patterns. |
| `INFO` | Informational only; no direct security impact. | Stylistic or advisory observations. |

---

## 2. Confidence levels

Confidence describes how likely a match is to be a **true positive** — i.e. the
precision of the rule, independent of severity.

| Level | Definition | Triage expectation |
| --- | --- | --- |
| `HIGH` | The pattern is precise and structurally unambiguous. | Few false positives; treat matches as real pending quick confirmation. |
| `MEDIUM` | Heuristic pattern; correct in the common case but context-dependent. | Manually confirm each match. |
| `LOW` | Broad / over-approximating pattern cast wide to avoid false negatives. | Expect to triage and dismiss false positives. |

Severity and confidence are **orthogonal**. A `CRITICAL` / `MEDIUM` rule means
"if real, this is catastrophic, but confirm the match"; a `MEDIUM` / `HIGH` rule
means "definitely present, moderate impact."

---

## 3. OWASP Smart Contract Top 10 (2025) mapping

Templates classify their `owasp` field against the **OWASP Smart Contract Top 10
(2025)** — the smart-contract-specific taxonomy, *not* the general web Top 10.

| Code | Category | Representative official templates |
| --- | --- | --- |
| `SC01:2025` | Access Control Vulnerabilities | Unprotected initializer, unrestricted `transferOwnership`, unprotected `selfdestruct`, unprotected ETH withdrawal |
| `SC02:2025` | Logic Errors | Delegatecall to user-controlled address, arbitrary low-level call, boolean constant misuse, `^` vs `**` misuse |
| `SC03:2025` | Reentrancy Attacks | Reentrancy (state write after external call), reentrancy via `balanceOf` delta |
| `SC04:2025` | Lack of Input Validation | Arbitrary `transferFrom`, delegatecall on unchecked target |
| `SC05:2025` | Integer Overflow & Underflow | Arithmetic in `unchecked` block, division-before-multiply |
| `SC06:2025` | Insecure Randomness | Weak randomness from block variables |
| `SC07:2025` | Unchecked External Calls | Unchecked low-level call / `send` / ERC20 transfer return value |
| `SC08:2025` | Denial of Service | `msg.value` in a loop, delegatecall in a loop |
| `SC09:2025` | Gas Limit Vulnerabilities | Loop-bound issues affecting gas |
| `SC10:2025` | Improper Upgradeability / misc. | Proxy storage layout collision, unprotected initializer |

Cross-cutting patterns may map to more than one category — pick the **primary**
risk the rule detects. Signature malleability, `tx.origin` authentication, and
hash-collision-via-`encodePacked` are commonly classified under access control
(`SC01`) or logic errors (`SC02`) depending on how they are abused.

> The `cwe` field (e.g. `CWE-841`, `CWE-829`, `CWE-682`) is complementary and
> may be set alongside `owasp` for tooling that consumes CWE identifiers.

---

## 4. Choosing values for a new rule

1. **Severity** — assume the match is real. What is the worst realistic outcome?
   Funds gone with no precondition → `CRITICAL`; serious-but-conditional →
   `HIGH`; smell / conditional → `MEDIUM`.
2. **Confidence** — how often will the pattern fire on safe code? Precise AST +
   dataflow constraint → `HIGH`; heuristic shape → `MEDIUM`; deliberately broad
   → `LOW`.
3. **OWASP** — map to the single primary SC Top 10 (2025) category above.

When in doubt, prefer the more conservative severity and the lower confidence,
and document the rationale in the PR.
