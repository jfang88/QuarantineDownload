# ADR-0015 — Controlled-data-release evidence-store boundary

- **Status:** Proposed
- **Date:** 2026-08-10
- **Owners:** Security Architecture, Platform Engineering, Data Governance
- **Affects:** Evidence database, lifecycle state, RBAC, retention, audit, reconciliation
- **Related:** [ADR-0003](0003-adopt-artifact-lifecycle-state-machine.md), [ADR-0013](0013-separate-ingress-and-data-release-control-planes.md), [`controlled-data-release-architecture.md`](../controlled-data-release-architecture.md)

## Context

ADR-0013 proposes separate sibling lifecycles for package/artifact ingress and controlled operational-data release while reusing shared platform primitives where semantics match. Both domains need canonical state, append-only transition history, immutable hashes, human/service identity, policy/evidence references, and audit integration.

However, the data stored in the two domains has materially different sensitivity and lifecycle semantics. Package-intake evidence records package coordinates, provenance, scanner results, approvals, repository state, vulnerabilities, and recall data. Controlled-data-release evidence records production source systems, file paths/queries, classifications, DLP/secrets findings, redaction lineage, destinations, vendor cases, transfer receipts, and purge/retention status. The latter can itself reveal sensitive operational information even when the released bytes are no longer retained.

A design decision is therefore required on whether both domains share the same tables/schema, share a database platform with bounded schemas, or use fully separate databases.

## Decision drivers

- Preserve ADR-0013's shared-platform efficiency without conflating lifecycle semantics.
- Prevent package users/operators from gaining access to sensitive release metadata by default.
- Support distinct retention and legal-hold rules for release evidence and bytes.
- Keep audit correlation possible across outbound vendor cases and inbound return files.
- Maintain independent schema evolution for two different state machines.
- Avoid a single overly broad service identity or database role spanning both domains.
- Keep backup/DR and operational support practical.

## Considered options

### Option A — One database and one shared schema/table model

Represent package-intake and data-release records in common workflow/evidence tables using a type discriminator.

**Pros**

- least initial engineering;
- easiest cross-domain reporting;
- one backup/DR and operational model.

**Cons**

- high risk of accidental RBAC/retention leakage between domains;
- schema becomes generic and difficult to reason about as lifecycle fields diverge;
- package operators may gain access to source paths, DLP findings, vendor cases, or other sensitive release metadata;
- changes for one lifecycle can create regressions in the other.

### Option B — Shared database platform with bounded sibling schemas and shared audit/identity primitives

Use the same managed PostgreSQL/database platform but separate package-intake and data-release schemas/table namespaces, service roles, migrations, lifecycle constraints, and retention policies. Share only deliberately defined primitives such as enterprise subject identifiers, request correlation IDs, common audit-event envelope, and cross-domain case references.

**Pros**

- retains operational reuse and common backup tooling;
- clearer RBAC, schema ownership, migration, and retention boundaries;
- supports cross-domain correlation without forcing a generic lifecycle;
- aligns closely to ADR-0013's "separate lifecycles, shared primitives" model.

**Cons**

- requires explicit schema and database-role design;
- shared platform outage can still affect both domains;
- cross-schema joins and shared reporting need controlled interfaces rather than ad hoc access.

### Option C — Fully separate database platforms

Run independent evidence databases for package intake and data release, integrating only through events or reporting APIs.

**Pros**

- strongest isolation and independent failure/maintenance domains;
- simplest data-classification boundary;
- independent retention, backup, and scaling policies.

**Cons**

- duplicates platform operations and DR;
- more integration required for vendor-return correlation and enterprise audit;
- higher cost and operational complexity.

## Proposed decision outcome

**Working recommendation: Option B — shared database platform with bounded sibling schemas and service roles. Decision remains Proposed.**

The data-release lifecycle should not reuse package-intake tables simply because both happen to store state transitions. Each domain should own its state machine, migrations, constraints, retention rules, and service identities. A narrow common audit envelope may be shared so enterprise SIEM/reporting can correlate events without granting either application broad direct access to the other's records.

## Consequences if accepted

- Data-release state and evidence use a separate schema/table namespace from package intake.
- Database roles and service identities are domain-specific; cross-domain reads require an explicit reporting/API path.
- Retention and legal-hold policies can differ between package evidence and operational-data-release evidence.
- Common request/case correlation IDs allow an outbound vendor case to be linked to a new inbound file-intake request without sharing lifecycle tables.
- Backup/DR can reuse the same database platform, but restoration tests must verify both schemas and their access policies independently.

## Follow-up actions

1. Define the common audit-event envelope and identifiers that may be shared safely.
2. Define separate database roles for package portal/workers and data-release portal/workers.
3. Add data-release schema migrations and lifecycle constraints to the POC once implementation begins.
4. Define retention and legal-hold behavior for release evidence separately from release bytes.
5. Prevent ad hoc cross-schema access by normal application service identities.
