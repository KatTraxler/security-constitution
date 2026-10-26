# Security Constitution Framework

A method for detecting **insecure-by-design** flaws in multi-platform services through
AI-assisted design review — architectural flaws that violate expected security
guarantees. This directory holds the taxonomy (Phase 1) and the
Security Constitution (Phase 2).

## What this contains

- **A Security Constitution** — first principles encoding secure design norms,
  operational lessons, the Anchors of Trust, the reasonable-customer test, and the
  six-condition deputy-graduation framework.
- **A Design Shape Register** — reusable trust-boundary shapes distilled from confirmed
  cases and the feature-abuse calibration set.
- **A Discovery Signal Register** — documentation/API clues that route review to shapes.
- **A Feature-Abuse Reference** — the negative calibration set (features that are
  not design flaws, used to guard against false positives).

This edition ships the shapes and signals behind the cases presented at the talk, "Your Harness Needs a Security Constitution", and the
feature-abuse negatives those shapes calibrate against. It is a reasoning artifact, not a
pipeline: it ships no model runner, scanner, or orchestration code — and it describes
*structure*, not specific findings.

## Source-of-Truth Map

| Artifact | Role |
|----------|------|
| `phase 1 - taxonomy/markdown/design-shape-register.md` | Reusable vulnerability structures and their confirmation criteria. |
| `phase 1 - taxonomy/markdown/discovery-signal-register.md` | Static clues that route review; signals are not findings by themselves. |
| `phase 1 - taxonomy/markdown/feature-abuse-reference.md` | Negative calibration corpus for powerful but adequately controlled features. |
| `phase 2 - security constitution/security-constitution.md` | Promptable principles, lessons, and evidence requirements. |

For the Phase 1 registers, the sibling `json/` directory is the machine-readable mirror.
Keep the JSON companion in sync whenever a semantic Markdown edit changes entries, labels,
or relationships; the two are meant to agree.

The method's cardinality: a confirmed case may map to many Shapes; Signals may route to many
Shapes. Conditional Signal routes remain suggestions until separately evidenced. The
constitution refers to canon entries (`DFC-###`) as abstract evidence labels; the canon
register itself is not part of this edition.

## Method discipline

These principles govern how a verdict is reached, independent of any particular tooling:

- **Hypothesize statically, validate by execution.** Constitution-only judgments are
  documentation signal, not findings; promotion requires live, execution-based corroboration.
- **The journal is the finding.** The per-case investigation record is the canonical
  record; a claim is supportable only to the extent that record supports it.
- **Try to break your own conclusions.** Run an adversarial-critic pass after every
  judgment, with the explicit instruction to falsify it.
- **Escalate from the cheapest tier.** Start validation at the lowest-effort check that can
  resolve the hypothesis; escalate only on inconclusive evidence.
- **Treat multipliers as evidence, not inheritance.** A severity multiplier is a potential
  test for a candidate, active only when the candidate carries independent evidence for the
  corresponding Signal.
