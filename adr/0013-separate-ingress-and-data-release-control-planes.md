# ADR-0013 — Separate ingress and operational-data release control planes

- **Status:** Proposed
- **Date:** 2026-08-10
- **Owners:** Security Architecture
- **Affects:** Repository scope, request workflow, lifecycle state, quarantine/staging, tooling, POC design
- **Related:** [ADR-0014](0014-controlled-data-release-sourcing-strategy.md), [ADR-0015](0015-data-release-evidence-store-boundary.md)

## Context

The repository originally focused on controlled software and file **ingress**: packages, libraries, binaries, installers, firmware, containers, and model artifacts are acquired from external sources, quarantined, analysed, approved, and promoted to internal repositories.

A second enterprise use case requires controlled movement of operational data such as logs, traces, crash dumps, packet captures, configuration extracts, database extracts, and support bundles from production/non-production systems to other internal networks or external vendor portals.

Although both use cases involve files, request/approval, quarantine, inspection, and audit, their security objectives are materially different:

- ingress primarily protects integrity and availability by preventing malicious or compromised external bytes from entering trusted environments;
- operational-data release primarily protects confidentiality and boundary integrity by preventing sensitive internal bytes from leaving an authorised trust zone or being delivered to an unauthorised destination;
- outbound transfer must additionally prevent the transfer mechanism from becoming a routable or privileged administrative/change channel;
- external vendor return content still requires an appropriate inbound quarantine/inspection path, selected by content type.

The architecture therefore needs an explicit decision on whether to merge both directions into one generic artifact lifecycle or keep them as separate but related control planes.

## Decision drivers

- Avoid confusing approval to acquire software with approval to disclose internal data.
- Preserve the strong package-intake lifecycle without overloading its states with unrelated outbound semantics.
- Make DLP, secrets detection, redaction, destination binding, and data minimisation first-class controls.
- Prevent cross-domain file transfer from becoming an administrative bridge or automated change channel.
- Reuse shared enterprise services such as identity, request workflow, evidence platform, object storage, malware scanning, and SIEM where semantics match.
- Keep POC implementation practical by sharing infrastructure while maintaining distinct lifecycle/state models.
- Ensure vendor-return content is explicitly routed to the appropriate inbound control rather than inheriting trust from an outbound support case.

## Considered options

### Option A — Treat logs and operational data as another package-intake artifact type

Use the existing package lifecycle and add a new artifact class such as `operational-data`.

**Pros**

- smallest apparent document and workflow change;
- maximum reuse of existing state machine and portal;
- single generic artifact record.

**Cons**

- conflates approval to fetch/promote with approval to collect/disclose;
- DLP, redaction, destination binding, and transfer receipts fit poorly into package states;
- package concepts such as approved repository, cooling-off, CVE recheck, and recall are not natural for released logs;
- increases the chance that an outbound approval is accidentally interpreted as permission to execute/apply returned content;
- makes security objectives less clear to reviewers and operators.

### Option B — Separate sibling control planes with shared platform primitives

Keep `package-intake-architecture.md` authoritative for ingress and add a separate controlled-data-release architecture and lifecycle. Reuse identity, portal, evidence platform, object storage, malware scanning, monitoring, and policy infrastructure where appropriate, while maintaining bounded lifecycle schemas and RBAC.

**Pros**

- clear security objectives and approval semantics;
- DLP/secrets/redaction/destination controls become first-class;
- package lifecycle remains coherent;
- shared infrastructure reduces duplicate engineering;
- vendor return path can explicitly cross back into the appropriate inbound quarantine/inspection control;
- easier to demonstrate `transfer != execute/change` as a hard architectural boundary.

**Cons**

- more documentation and lifecycle logic;
- portal/evidence platform needs a second workflow type and bounded schema/service roles;
- shared services require careful namespace and RBAC separation;
- tooling/procurement analysis must cover an additional capability domain.

### Option C — Build a completely independent data-release platform

Treat operational-data release as unrelated to package intake and deploy separate identity/workflow/evidence/storage/tooling.

**Pros**

- strong isolation;
- each platform can be independently selected and operated;
- simpler component-level trust boundaries.

**Cons**

- duplicates identity, workflow, audit, storage, policy, operations, and engineering;
- inconsistent user experience and evidence models;
- higher cost and operational burden;
- harder to correlate outbound support cases with vendor-return ingress.

## Proposed decision outcome

**Proposed: Option B — separate sibling control planes with shared platform primitives.**

The repository should contain:

- `package-intake-architecture.md` for external software/artifact ingress;
- `controlled-data-release-architecture.md` for operational-data egress/cross-domain release;
- `controlled-data-release-tooling.md` for implementation/procurement capability mapping;
- `controlled-data-release-poc.md` for the POC extension and demonstration;
- shared ADR, identity, evidence-platform, storage, monitoring, and workflow primitives where reuse is safe;
- separate lifecycle semantics and bounded evidence access for each domain.

This ADR remains `Proposed` until formally accepted in architecture review. The new documents are therefore draft reference material and do not claim the production boundary decision is ratified.

The two immediate downstream decisions are intentionally separated from this structural ADR:

- [ADR-0014](0014-controlled-data-release-sourcing-strategy.md) — Build/Buy/Hybrid sourcing for the data-release capability;
- [ADR-0015](0015-data-release-evidence-store-boundary.md) — shared tables vs. bounded sibling schemas vs. separate evidence databases.

## Consequences if accepted

### Positive

- package intake remains focused on supply-chain integrity;
- outbound data release gains explicit confidentiality and destination controls;
- source collection, transfer, and privileged change are cleanly separated;
- the POC can demonstrate both dangerous-data-in and sensitive-data-out scenarios using shared infrastructure;
- vendor-return content has an unambiguous type-based route into inbound quarantine/inspection;
- procurement can evaluate Managed File Transfer/cross-domain capability separately from repository/package-security products.

### Negative / costs

- additional lifecycle/state model must be implemented;
- request portal and evidence platform must support multiple workflow types with bounded access;
- DLP/secrets/redaction and transfer-broker capabilities add new technology dependencies;
- more controls require ownership decisions, especially data classification, destination policy, retention, preservation, and emergency release.

## Follow-up actions

1. Review and accept/reject this ADR.
2. Review [ADR-0014](0014-controlled-data-release-sourcing-strategy.md) for the data-release Build/Buy/Hybrid direction.
3. Review [ADR-0015](0015-data-release-evidence-store-boundary.md) for the lifecycle/evidence schema boundary.
4. Define authoritative DLP/data-classification policy and ownership.
5. Define approved destination classes, vendor onboarding, and secure human-assisted upload requirements.
6. Define retention/purge and preservation/legal-hold requirements for original, transformed, and transferred copies.
7. Define emergency/break-glass release controls.
8. Ensure generic vendor-return documents/data have a secure inbound quarantine/content-inspection path rather than being forced through package promotion or trusted by association.
9. Implement and test the controlled-data-release POC acceptance criteria before treating the control model as proven.
