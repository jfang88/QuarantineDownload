# Architecture Decision Records

This directory tracks the decisions behind the repository's controlled file-movement architectures — both settled ones and ones still open for debate. It began with the package-intake architecture and now also covers the proposed controlled operational-data release / cross-domain transfer control plane.

The ADR register replaces inline "Open issues for internal review" sections as the canonical place for **architecture and sourcing decisions**. Not every policy parameter needs its own ADR: retention periods, data-classification rules, redaction standards, and similar organization-specific policy inputs may be governed in policy/standards documents instead. An ADR is a single, dated, append-only record per architectural decision — when a decision changes, a new ADR supersedes the old one rather than editing history in place.

## Format

Each ADR follows a lightweight MADR-style template: Status, Context, Decision Drivers, Considered Options (with pros/cons for each), Decision Outcome, and Consequences. See any file below for the pattern.

Most ADRs here are organizational/process judgment calls (which enforcement mechanism, which cadence, which registry) rather than external factual claims, so they don't carry their own References section. Where an ADR does rely on a specific external fact — for example, ADR-0004's VirusTotal Public API terms — that fact is verified in the citing main document's own **References** section rather than duplicated here. ADR-0012 is the exception: it summarizes `enterprise-build-vs-buy-evaluation.md`, which carries its own extensive references.

**Status values used in this directory:**

| Status | Meaning |
|---|---|
| `Proposed` | Options are laid out with pros/cons; no decision has been made yet. Needs a review-meeting sign-off. |
| `Accepted` | A decision has been made and is reflected in the architecture/tooling documents. |
| `Superseded by ADR-XXXX` | An earlier decision was later replaced; the new ADR is the current one. |

## Index

**Status at a glance: 12 open (need a decision) · 3 accepted · 0 superseded.** Open items are listed first so they don't get lost among settled ones — this table is the answer to "what's still undecided" at the architecture/sourcing level without having to open every file.

The original nine open decisions remain primarily scoped to `package-intake-architecture.md` and `solution-architecture-tooling.md`. ADR-0013, ADR-0014, and ADR-0015 cover the repository-level/data-release additions: the relationship between the control planes, the data-release sourcing strategy, and the data-release evidence-store boundary.

### 🟡 Open — needs a review-meeting decision

| ADR | Title | Affects |
|---|---|---|
| [0004](0004-virustotal-public-api-usage-and-licensing.md) | VirusTotal Public API usage and paid-tier procurement trigger | Stage 11b recheck capacity/licensing |
| [0005](0005-push-remediation-coverage-boundary.md) | Push-remediation coverage boundary for unmanaged endpoints | Stage 10 recall reach |
| [0006](0006-segregation-of-duties-enforcement-mechanism.md) | Segregation-of-duties enforcement mechanism for pipeline administration | Pipeline-admin access control |
| [0007](0007-model-registry-tooling-choice.md) | Model registry tooling: Nexus tags + CMDB vs. a dedicated platform | Path C / Stage 5c |
| [0008](0008-export-control-and-legal-review-routing.md) | Export control and legal review routing | Stage 1 intake |
| [0009](0009-alert-tuning-ownership-and-cadence.md) | Alert-tuning ownership and cadence | Stage 11b false-positive management |
| [0010](0010-control-id-taxonomy-and-traceability-matrix.md) | Normative control-ID taxonomy and traceability matrix | Whole package-intake architecture document — explicitly deferred, not just unprioritized |
| [0011](0011-backup-rpo-rto-targets.md) | Backup RPO/RTO targets and tooling for systems of record | Nexus, CMDB, recheck datastore |
| [0012](0012-build-vs-buy-vs-hybrid-sourcing-strategy.md) | Build-versus-buy-versus-hybrid sourcing strategy for enterprise artifact ingress | **Ingress domain only** — package/file ingress sourcing direction; does not decide controlled-data-release sourcing |
| [0013](0013-separate-ingress-and-data-release-control-planes.md) | Separate ingress and operational-data release control planes | **Repository structure** — whether package/software ingress and operational-data egress/cross-domain release remain separate sibling lifecycles sharing platform primitives |
| [0014](0014-controlled-data-release-sourcing-strategy.md) | Controlled-data-release sourcing strategy | Build/buy/hybrid direction for controlled collection, DLP/content inspection, redaction, MFT/transfer brokering, and release operations |
| [0015](0015-data-release-evidence-store-boundary.md) | Controlled-data-release evidence-store boundary | Whether data release shares package tables, uses bounded sibling schemas on a shared DB platform, or uses a separate evidence database |

### 🟢 Decided

| ADR | Title | Status |
|---|---|---|
| [0001](0001-two-document-split.md) | Keep the architecture and tooling guide as two separate documents | Accepted |
| [0002](0002-adopt-architecture-decision-records.md) | Adopt Architecture Decision Records in place of inline open issues | Accepted |
| [0003](0003-adopt-artifact-lifecycle-state-machine.md) | Adopt a formal artifact lifecycle state machine | Accepted |

## Architecture documents covered by this register

### Package/software ingress

- [`../package-intake-architecture.md`](../package-intake-architecture.md)
- [`../solution-architecture-tooling.md`](../solution-architecture-tooling.md)
- [`../enterprise-build-vs-buy-evaluation.md`](../enterprise-build-vs-buy-evaluation.md)

### Controlled operational-data release

- [`../controlled-data-release-architecture.md`](../controlled-data-release-architecture.md)
- [`../controlled-data-release-tooling.md`](../controlled-data-release-tooling.md)
- [`../controlled-data-release-poc.md`](../controlled-data-release-poc.md)

## Adding a new ADR

1. Copy the template structure from any existing file.
2. Number it sequentially (next unused four-digit number).
3. Set `Status: Proposed` and lay out the considered options with honest pros/cons — do not pre-select a winner in the Context section.
4. Add it to the index table above.
5. When a decision is made, update `Status` to `Accepted`, fill in the Decision Outcome section, move the row from "🟡 Open" to "🟢 Decided" in the index above (updating the open/accepted counts in the status line), and reflect the decision in the relevant architecture/tooling document section (with a cross-reference back to the ADR).
6. If a later decision reverses an earlier one, do not edit the old ADR's decision — create a new ADR and mark the old one `Superseded by ADR-XXXX`.
