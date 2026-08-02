# Architecture Decision Records

This directory tracks the decisions behind the package intake architecture — both settled ones and ones still open for debate. It replaces the inline "Open issues for internal review" sections that previously lived at the bottom of [`package-intake-architecture.md`](../package-intake-architecture.md) and [`solution-architecture-tooling.md`](../solution-architecture-tooling.md): those sections conflated undecided questions with normative document content, and once a decision was made there was no clean record of *which* option was chosen and why. An ADR is a single, dated, append-only record per decision — when a decision changes, a new ADR supersedes the old one rather than editing history in place.

## Format

Each ADR follows a lightweight MADR-style template: Status, Context, Decision Drivers, Considered Options (with pros/cons for each), Decision Outcome, and Consequences. See any file below for the pattern.

**Status values used in this directory:**

| Status | Meaning |
|---|---|
| `Proposed` | Options are laid out with pros/cons; no decision has been made yet. Needs a review-meeting sign-off. |
| `Accepted` | A decision has been made and is reflected in the architecture/tooling documents. |
| `Superseded by ADR-XXXX` | An earlier decision was later replaced; the new ADR is the current one. |

## Index

**Status at a glance: 8 open (need a decision) · 3 accepted · 0 superseded.** Open items are listed first so they don't get lost among settled ones — this table is the answer to "what's still undecided" without having to open every file. It is the single register for both `package-intake-architecture.md` and `solution-architecture-tooling.md`; neither document repeats this table, they only link to it (see "Open decisions" in each).

### 🟡 Open — needs a review-meeting decision

| ADR | Title | Affects |
|---|---|---|
| [0004](0004-virustotal-public-api-usage-and-licensing.md) | VirusTotal Public API usage and paid-tier procurement trigger | Stage 11b recheck capacity/licensing |
| [0005](0005-push-remediation-coverage-boundary.md) | Push-remediation coverage boundary for unmanaged endpoints | Stage 10 recall reach |
| [0006](0006-segregation-of-duties-enforcement-mechanism.md) | Segregation-of-duties enforcement mechanism for pipeline administration | Pipeline-admin access control |
| [0007](0007-model-registry-tooling-choice.md) | Model registry tooling: Nexus tags + CMDB vs. a dedicated platform | Path C / Stage 5c |
| [0008](0008-export-control-and-legal-review-routing.md) | Export control and legal review routing | Stage 1 intake |
| [0009](0009-alert-tuning-ownership-and-cadence.md) | Alert-tuning ownership and cadence | Stage 11b false-positive management |
| [0010](0010-control-id-taxonomy-and-traceability-matrix.md) | Normative control-ID taxonomy and traceability matrix | Whole architecture document — explicitly deferred, not just unprioritized |
| [0011](0011-backup-rpo-rto-targets.md) | Backup RPO/RTO targets and tooling for systems of record | Nexus, CMDB, recheck datastore |

### 🟢 Decided

| ADR | Title | Status |
|---|---|---|
| [0001](0001-two-document-split.md) | Keep the architecture and tooling guide as two separate documents | Accepted |
| [0002](0002-adopt-architecture-decision-records.md) | Adopt Architecture Decision Records in place of inline open issues | Accepted |
| [0003](0003-adopt-artifact-lifecycle-state-machine.md) | Adopt a formal artifact lifecycle state machine | Accepted |

## Adding a new ADR

1. Copy the template structure from any existing file.
2. Number it sequentially (next unused four-digit number).
3. Set `Status: Proposed` and lay out the considered options with honest pros/cons — do not pre-select a winner in the Context section.
4. Add it to the index table above.
5. When a decision is made, update `Status` to `Accepted`, fill in the Decision Outcome section, move the row from "🟡 Open" to "🟢 Decided" in the index above (updating the open/accepted counts in the status line), and reflect the decision in the relevant architecture/tooling document section (with a cross-reference back to the ADR).
6. If a later decision reverses an earlier one, do not edit the old ADR's decision — create a new ADR and mark the old one `Superseded by ADR-XXXX`.
