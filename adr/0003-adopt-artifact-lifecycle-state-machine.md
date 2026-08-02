# ADR-0003: Adopt a formal artifact lifecycle state machine

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-08-02 |
| Deciders | Security Architecture |
| Related | `package-intake-architecture.md` § "Artifact lifecycle state machine", Stage 3 "Per-artifact evidence database" |

## Context

The architecture describes an artifact's journey — request, fetch, analysis, cooling-off, testing, promotion, and post-approval monitoring — as a sequence of numbered stages (1 through 11b), and represents *some* of that journey concretely as Nexus repository-group membership (`intake-quarantine` → `intake-dev-approved` → `intake-prod-approved`). The independent review pointed out that repository-group membership was never designed to be a complete state machine: it has no representation for "analysis is inconclusive and awaiting human disposition," "test failed," "suspended pending recheck investigation but not yet recalled," or "approval expired but artifact bytes are unchanged and eligible for renewal without re-fetch." Those states exist in the prose description of the stages but not as an explicit, queryable, transition-controlled model, which makes two things hard: knowing an artifact's exact current status without re-deriving it from scattered signals across GitLab/Nexus/CMDB, and reasoning about which transitions are actually valid (can a `SUSPENDED` artifact go straight back to `APPROVED`, or does it have to go through `ANALYSING` again?).

## Decision drivers

- Need a single, queryable answer to "what state is this artifact in right now" that doesn't require re-deriving status from repository-group membership plus GitLab issue labels plus CMDB fields plus tribal knowledge of which stage number implies which status.
- Need explicit, enumerated valid transitions so a pipeline bug (or a manual intervention) can't put an artifact into a nonsensical state (e.g., `RECALLED` artifact silently reappearing as `APPROVED`).
- Need idempotency and retry semantics defined once, centrally, rather than reimplemented ad hoc in each stage's pipeline script.
- Should integrate naturally with the per-artifact evidence database already introduced to fix the Nexus-tagging-API misuse identified elsewhere in this repository's review response — the state machine and the evidence schema are two views of the same underlying record, and are cheapest to design together.

## Considered options

### Option A: Leave the lifecycle implicit in the stage numbering and Nexus repository groups

**Pros**
- No new schema or design work.
- Matches the current implementation exactly — nothing to migrate.

**Cons**
- Does not solve any of the problems in Context; this is the status quo being replaced, not an alternative.
- Repository-group membership actively under-represents the real state space (no group for "inconclusive," "suspended," "test failed," etc.), so states that matter operationally are invisible to anything that only reads Nexus.

### Option B: Adopt an explicit state machine with a canonical authoritative store

**Pros**
- Makes every reachable state and every valid transition explicit and enumerable — closes the "can this actually happen" ambiguity the review flagged.
- Gives a natural home for idempotency keys and retry rules, defined once rather than per pipeline script.
- Pairs naturally with the evidence database: the database's transition log becomes the append-only source of truth, with GitLab labels, Nexus repository-group membership, and CMDB status fields as derived reflections that a reconciliation job can check for drift.
- Directly answers the review's specific critique that repository groups were being treated as a de facto state machine they were never designed to be.

**Cons**
- Real design and implementation cost: a new table/schema, transition-writing logic in every pipeline stage, and a reconciliation job to keep the reflected copies (GitLab, Nexus, CMDB) in sync with the canonical store.
- Adds a dependency: every pipeline stage script now needs to write a state transition, not just perform its stage-specific work — a missed transition write is a new failure mode that didn't exist when state was informally inferred.

### Option C: Adopt a state machine, but represent it entirely through existing tools' native state features (GitLab issue workflow states plus Nexus repository groups) instead of a new schema

**Pros**
- No new datastore to build or back up.

**Cons**
- Neither GitLab issue states nor Nexus repository groups were built to express the full state space this architecture needs (see Option A's cons) — extending them to fit would mean overloading GitLab labels and inventing more Nexus repository groups than the tool comfortably supports, which is a worse fit than a purpose-built schema and doesn't actually avoid new design work, just spreads it across two tools that weren't meant for it.

## Decision outcome

**Chosen option: B.** The state machine is defined in `package-intake-architecture.md` under "Artifact lifecycle state machine," using the evidence database (Stage 3) as the canonical, authoritative transition log, with GitLab, Nexus, and CMDB treated as reflected views reconciled against it.

## Consequences

- `package-intake-architecture.md` gains a new normative section defining every state, valid transition, triggering actor, and required evidence.
- The evidence database schema (already required per the Repository Firewall / evidence store correction) must include an append-only transition log, not just a current-state field, to support the audit and idempotency requirements.
- A reconciliation job is required to detect drift between the canonical state and its reflections in GitLab/Nexus/CMDB — this is new operational surface, not a one-time build cost.
- Every pipeline stage script gains a responsibility it didn't have before: writing a state transition with an idempotency key, not just performing its stage's work.
