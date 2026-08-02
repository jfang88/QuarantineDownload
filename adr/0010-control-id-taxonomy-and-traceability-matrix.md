# ADR-0010: Normative control-ID taxonomy and traceability matrix

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-08-02 |
| Deciders | Security Architecture |
| Related | `package-intake-architecture.md` (all stages); ADR-0002, ADR-0003 |

## Context

The independent review recommended introducing stable control IDs (`INTAKE-*`, `FETCH-*`, `AUTH-*`, `ANALYSIS-*`, `PROMOTE-*`, `CONSUME-*`, `MONITOR-*`, `RECALL-*`, `PLATFORM-*`) and RFC-style requirement language (MUST/SHOULD/MAY), with each control formally mapped to threat, owner, implementation, evidence, failure mode, metric, and test. This is unambiguously good practice for a document that needs to pass external audit or formal compliance review — it is also a substantially larger and more rigid authoring effort than the prose-plus-tables format the architecture document currently uses, and is a poor fit to retrofit incrementally control-by-control. Unlike ADR-0002 and ADR-0003, which were adopted as part of the same review-response pass that identified them, this one was explicitly deferred rather than actioned, because it's the one item where the right answer depends on a fact about the document's intended use that hasn't been established.

## Decision drivers

- Whether this repository's purpose is an internal reference architecture (where prose control descriptions are adequate) versus a document that must pass external audit or formal compliance review (where a control-ID/traceability matrix is close to a requirement).
- Authoring and maintenance cost: a full control catalogue is a meaningfully larger and more rigid document than the current stage-based prose structure, and harder to keep current as controls evolve.
- This is a poor candidate for incremental/partial adoption — a traceability matrix that covers half the controls is arguably worse than none, since it implies false completeness for the controls it omits.

## Considered options

### Option A: Do not adopt a control-ID taxonomy; keep the current stage-based prose-and-tables structure

**Pros**
- No new authoring or maintenance burden.
- Matches the document's current purpose as an internal reference architecture, if that remains its purpose.

**Cons**
- If the document's purpose later shifts toward audit/compliance use, this becomes a larger retrofit later than if control IDs had been designed in from an earlier point.
- Individual controls are harder to reference precisely (by ID) from ADRs, tickets, or test plans without one — this ADR itself has to reference controls by section name/stage number instead of a stable ID, which is exactly the friction a control-ID scheme would remove.

### Option B: Adopt the full control-ID taxonomy and traceability matrix now

**Pros**
- Makes the architecture audit-ready and gives every control a stable, referenceable identity.
- Forces an explicit MUST/SHOULD/MAY classification per control, which surfaces ambiguity in the current prose (some "must" language today may really be "should" once forced to choose).

**Cons**
- Large authoring effort across every stage of an already-large document, undertaken before it's confirmed the document needs audit-grade rigor.
- Ongoing maintenance cost compounds with every future control change — a smaller documentation change today becomes a larger one once every prose edit must also update a matrix entry.

### Option C: Adopt a control-ID taxonomy for new controls only, going forward, without retrofitting existing ones

**Pros**
- Bounds the immediate cost — no large retrofit project.
- Establishes the pattern for future controls, partially preparing for a later full adoption if needed.

**Cons**
- Produces a document with inconsistent rigor — some controls have IDs and traceability data, most don't — which the Context section already flags as arguably worse than a consistent absence, since it implies the ID'd controls are the complete/audited set.

## Decision outcome

*Undecided — this is explicitly deferred, not merely unprioritized.* The deciding fact — whether this document needs to serve an external audit or compliance function — has not been established. This ADR should be revisited when that fact changes, not on a fixed schedule.

## Consequences

*None while `Proposed`* — the architecture document keeps its current stage-based structure. If this is later `Accepted`, expect it to be one of the largest single editing efforts applied to `package-intake-architecture.md` to date, and consider pairing it with ADR-0003's lifecycle state machine (transition evidence requirements map naturally onto the "evidence" column a control-ID matrix would need).
