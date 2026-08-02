# ADR-0008: Export control and legal review routing

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-08-02 |
| Deciders | Security Architecture, Legal/Export Control |
| Related | `package-intake-architecture.md` § Stage 1, § Stage 9; `solution-architecture-tooling.md` § Export control routing |

## Context

Neither main document currently routes firmware, cryptographic libraries, or certain commercial software through an export-control (ITAR/EAR) or legal review gate. This may already be handled by an existing legal process entirely outside this architecture's scope, or it may be a genuine gap — that fact is not currently known and is the actual open question, more than which of the two options below is preferable in the abstract.

## Decision drivers

- Avoid building duplicate classification logic into this pipeline if an existing legal process already covers artifacts entering through it.
- Avoid a silent compliance gap if no such process exists and cryptographic/firmware artifacts are flowing through intake unreviewed.

## Considered options

### Option A: Confirm an existing legal/export-control process already covers artifacts entering through this pipeline, and add a pointer/webhook from Stage 1 intake into that process rather than duplicating it

**Pros**
- Cheapest integration if the process already exists — a webhook, not new classification logic.
- Avoids two parallel, potentially inconsistent export-control determinations for the same artifact.

**Cons**
- Depends entirely on such a process existing and being willing/able to accept a webhook trigger from this pipeline — not yet confirmed.

### Option B: If no such process exists, add an explicit export-control classification field at Stage 1 intake and a legal-review gate before Stage 9 promotion for artifact categories that require it

**Pros**
- Closes the gap directly if no existing process is available to hook into.
- Classification field at intake gives visibility into which artifact categories actually need review, which is useful data regardless of the final routing decision.

**Cons**
- Requires building classification logic and a review gate from scratch, including defining who has legal-review authority and what the gate's SLA is — meaningfully more work than Option A.

## Decision outcome

*Undecided — pending confirmation of whether an existing legal/export-control process already applies to this pipeline's artifacts.* This confirmation is a prerequisite fact-finding step, not itself a design choice — until it's answered, neither option can be responsibly selected over the other.

## Consequences

*To be completed once a decision is recorded.* Whoever confirms (or rules out) an existing process should record the answer here even if the ADR status doesn't move to `Accepted` yet, since "we checked and no such process exists" is itself useful decision-relevant information worth preserving.
