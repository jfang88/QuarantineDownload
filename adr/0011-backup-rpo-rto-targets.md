# ADR-0011: Backup RPO/RTO targets and tooling for systems of record

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-08-02 |
| Deciders | Security Architecture, Infrastructure/Platform |
| Related | `package-intake-architecture.md` § Backup and recovery for systems of record; `solution-architecture-tooling.md` § Supporting infrastructure tools (pgBackRest/restic) |

## Context

Nexus, the CMDB, and the recheck job datastore are systems of record, not caches — losing them loses the evidence base for every approval, advisory, and recheck verdict this architecture depends on, which matters most exactly when an incident is already in progress. The architecture requires "an explicit backup and recovery posture" for each but does not set specific Recovery Point Objective (RPO) or Recovery Time Objective (RTO) numbers, or commit to specific backup tooling beyond noting `pgBackRest`/`restic` as candidates. Those numbers are an organizational risk-tolerance and infrastructure-budget decision, not something this architecture should assume on the organization's behalf.

## Decision drivers

- RPO/RTO targets should reflect actual organizational risk tolerance for losing recent approval/recheck history, not an arbitrary default.
- The recheck job datastore in particular is primary audit evidence during a post-approval compromise investigation — its retention window should match or exceed the incident-investigation lookback requirement, which is itself an organizational policy input this architecture doesn't set.
- Backup tooling choice should fit whatever infrastructure/backup platform the organization already operates, rather than introducing a new one solely for this pipeline.

## Considered options

### Option A: Set uniform RPO/RTO targets across all three systems of record (e.g., RPO 24h, RTO 4h) and standardize on `pgBackRest`/`restic` as proposed

**Pros**
- Simple, one policy to communicate and audit.
- Matches the tooling already named as a candidate in the tooling guide.

**Cons**
- Treats all three systems as equally critical, when the recheck job datastore's audit-evidence role arguably warrants a longer retention window (matching incident-investigation lookback) than the operational RPO/RTO numbers that make sense for Nexus's day-to-day availability.
- Assumes `pgBackRest`/`restic` fits the organization's existing infrastructure without confirming that.

### Option B: Set differentiated targets per system, reflecting each one's actual role (Nexus: availability-driven RPO/RTO; CMDB: availability-driven; recheck datastore: retention-driven, sized to the incident-investigation lookback policy)

**Pros**
- More accurately reflects that "how fast do we need this back" (RTO) and "how long do we need to keep history" (retention) are different questions with different right answers per system.
- Recheck datastore retention explicitly ties to a real organizational requirement (incident-investigation lookback) rather than an arbitrary number.

**Cons**
- More decisions to make and document than a single uniform policy.
- Requires the incident-investigation lookback policy to actually exist and be known — if it doesn't, this option has a prerequisite fact-finding step similar to ADR-0008's export-control question.

### Option C: Defer specific numbers; require only that each system's backup be tested (a documented restore drill) before go-live, without committing to numeric RPO/RTO targets in this ADR

**Pros**
- Avoids picking numbers without the risk-tolerance/infrastructure-budget input needed to pick them well.
- A tested restore is valuable regardless of what the eventual numeric targets are.

**Cons**
- Doesn't actually close the open question — just defers it again, with a weaker forcing function than a numeric target would provide.

## Decision outcome

*Undecided — pending input from Infrastructure/Platform on existing backup tooling and organizational RPO/RTO risk tolerance, and from Security on the incident-investigation lookback policy that should size the recheck datastore's retention window.* Option B is the most defensible design if those inputs are available; Option C is a reasonable interim step if they are not yet available at decision time.

## Consequences

*To be completed once a decision is recorded.* At minimum, whichever option is chosen should result in a documented, tested restore procedure for all three systems before Phase 1 rollout is considered complete, per the tooling guide's phased implementation plan.
