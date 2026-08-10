# ADR-0014 — Controlled-data-release sourcing strategy

- **Status:** Proposed
- **Date:** 2026-08-10
- **Owners:** Security Architecture, Procurement, Operations
- **Affects:** `controlled-data-release-tooling.md`, MFT/transfer broker, DLP/content inspection, redaction, controlled collection, operational support model
- **Related:** [ADR-0013](0013-separate-ingress-and-data-release-control-planes.md), [`controlled-data-release-architecture.md`](../controlled-data-release-architecture.md), [`controlled-data-release-tooling.md`](../controlled-data-release-tooling.md)

## Context

The controlled-data-release architecture introduces a sibling control plane for moving logs, traces, diagnostic bundles, crash dumps, packet captures, database extracts, configuration exports, and related operational data from protected environments to other trust zones or external support parties.

Its implementation requires capabilities that are different from package/artifact ingress: controlled source collection, private content inspection, secrets detection, DLP/data classification, redaction/minimisation, destination binding, Managed File Transfer or equivalent transfer brokering, transfer receipts, and release-specific purge/retention.

`controlled-data-release-tooling.md` currently describes a **hybrid** pattern as the likely or preferred enterprise direction. That is useful architectural analysis, but it must not be mistaken for a ratified sourcing decision. This ADR provides the same governance boundary for data release that ADR-0012 provides for artifact ingress.

## Decision drivers

- Reuse of existing enterprise ITSM, identity, DLP, secure web gateway, MFT, object storage, PAM, and SIEM investments.
- Ability to process highly sensitive production data privately without unintended submission to public or third-party services.
- Strength of destination administration, protocol handling, receipts, retry/idempotency, HA/DR, and partner support.
- Quality of DLP, secrets detection, archive processing, redaction, and large-file handling.
- Engineering capacity to build and operate lifecycle, evidence, policy, collection, transformation, and transfer integrations.
- Support for disconnected, highly segmented, sovereignty-sensitive, or regulated environments.
- Need to prevent a transfer platform from becoming a general network bridge or privileged change channel.
- Total cost of ownership, including specialist operations and vendor licensing.

## Considered options

### Option A — Build the data-release platform from existing and open components

Assemble existing enterprise identity/workflow with custom collectors, object storage, scanners, DLP/secrets integrations, redaction workers, destination connectors, evidence storage, and policy orchestration.

**Pros**

- maximum control over data location and processing;
- strong fit for disconnected or highly bespoke environments;
- can reuse existing internal platforms and avoid a new large product footprint;
- policy and evidence model can exactly match enterprise requirements.

**Cons**

- high custom engineering and testing burden;
- difficult protocol, partner, browser-portal, retry, HA/DR, and transfer-operations work;
- organization owns DLP detector quality, redaction correctness, lifecycle integrity, and support indefinitely;
- easy to underestimate operational cost and edge cases around large/complex files.

### Option B — Buy an integrated secure file-transfer / data-release platform

Procure one or more commercial products intended to cover secure exchange, workflow, content inspection, DLP, partner endpoints, and transfer operations.

**Pros**

- faster time to mature transfer operations;
- supported protocols, destination profiles, receipts, HA/DR, and partner onboarding;
- commercial content-security and DLP integrations may reduce custom engineering;
- contractual vendor support and upgrade path.

**Cons**

- no assumption that one product fully covers source minimisation, enterprise-specific approvals, redaction lineage, canonical release state, or all data types;
- licence and data-processing costs may be high;
- cloud-dependent inspection may conflict with confidentiality, sovereignty, or air-gap requirements;
- product workflow can become the de facto architecture unless evidence/state portability is deliberately preserved.

### Option C — Hybrid: buy/reuse specialist capabilities and retain a thin enterprise control plane

Reuse enterprise identity/ITSM/policy foundations, buy or reuse mature DLP/content-security and MFT capabilities, and keep enterprise ownership of release identity, lifecycle state, evidence correlation, destination/purpose binding, exception policy, transformation lineage, and source/destination integrations.

**Pros**

- avoids rebuilding mature transfer engines and DLP capabilities;
- preserves enterprise control over the security decision and evidence model;
- allows different products for different network classifications or sovereignty zones;
- aligns with ADR-0013's shared-platform-primitives model without forcing package and data-release lifecycles together.

**Cons**

- integration remains substantial;
- combines licence cost with internal engineering cost;
- requires disciplined ownership boundaries so bought products and custom orchestration do not duplicate each other;
- multiple products can create fragmented operations if canonical state/evidence are not clearly defined.

## Proposed decision outcome

**Working recommendation: Option C — Hybrid. Decision remains Proposed.**

The architecture should assume enterprise identity, request/approval policy, canonical release state, evidence correlation, and destination/purpose binding remain organization-controlled. Mature private DLP/content-inspection, MFT/secure exchange, and specialist redaction capabilities should be bought or reused where they meet security and sovereignty requirements rather than rebuilt without a strong reason.

Before accepting this ADR, the organization should inventory existing strategic products and run representative proof-of-capability scenarios using clean logs, sensitive logs, large archives, PCAP/dump handling, vendor portal upload, disconnected transfer, destination substitution, hash substitution, and purge/retention controls.

## Consequences if accepted

- `controlled-data-release-tooling.md` becomes a capability/integration guide rather than a product-selection document.
- Exact DLP, MFT, secure-browser, redaction, or cross-domain products are selected through downstream product ADRs or procurement decisions.
- The enterprise retains a canonical release state/evidence model even when commercial products implement individual stages.
- Private/on-premises processing remains a mandatory evaluation dimension for sensitive production data.
- POC components remain demonstration substitutes and are not assumed to be production selections.

## Follow-up actions

1. Inventory existing MFT, DLP, secure web/browser, PAM, ITSM, object-storage, and SIEM capabilities.
2. Define representative transfer volumes, maximum file sizes, data classifications, destination types, and disconnected-network requirements.
3. Run vendor/product demonstrations against the scenarios in `controlled-data-release-poc.md` plus production-scale large-file tests.
4. Decide exact product boundaries only after this sourcing-strategy ADR is accepted.
5. Link product-specific decisions back to this ADR and ADR-0013.
