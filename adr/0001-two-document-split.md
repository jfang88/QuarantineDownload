# ADR-0001: Keep the architecture and tooling guide as two separate documents

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-08-02 |
| Deciders | Security Architecture |
| Related | [`package-intake-architecture.md`](../package-intake-architecture.md), [`solution-architecture-tooling.md`](../solution-architecture-tooling.md), ADR-0002 |

## Context

The repository carries two large documents that cover overlapping ground: `package-intake-architecture.md` (control model, stages, control objectives, data flows) and `solution-architecture-tooling.md` (specific tools, licensing, deployment commands, phased rollout). Several stage descriptions exist in both documents in slightly different words, and every correction made against the independent review (`architecture-tooling-review.md`) had to be applied twice — once per document — which is real, observed duplication-and-drift risk, not a hypothetical one. The question is whether to merge them into a single document or keep the split, and if kept, how to reduce the duplication cost.

## Decision drivers

- Risk of the two documents drifting out of sync when one is updated and the other isn't (observed directly during the review-response pass).
- Different intended audiences: the architecture document is written for security/governance sign-off; the tooling document is written for the engineers implementing it.
- Different rate of change: control decisions (what must be true) should be stable; tool choices, versions, and vendor pricing (how it's achieved) change far more often.
- Reviewability: a combined document covering both control rationale and command-line implementation detail would run well past 2,000 lines, which is hard to review as a single diff or sign off on as a single artifact.
- Navigability for a first-time reader who wants "the whole picture" in one pass.

## Considered options

### Option A: Merge into a single document

**Pros**
- One file to read, one table of contents, no cross-document jumping.
- Eliminates the duplication-drift risk entirely — there is only one place to fix a factual error.
- Simpler repository structure.

**Cons**
- Conflates two audiences and two review cadences into one document: a security reviewer approving the control model would also be asked to approve (or at least wade through) Dockerfile snippets and Postgres connection strings, and vice versa.
- A tool swap (e.g., Nexus → Artifactory, or a VirusTotal tier change) would touch the same document a security-approved control model lives in, creating pressure to either re-review the whole document on every tooling change or let tooling changes slip through without the scrutiny a control change gets.
- A single file at the combined size (~2,000+ lines) is harder to review as one diff, and GitHub's diff view degrades in usefulness well before that size.

### Option B: Keep two documents, tighten cross-referencing to reduce duplication

**Pros**
- Preserves the audience and change-velocity separation: architecture stays product-neutral and stable; tooling absorbs the churn of version bumps, pricing changes, and command syntax updates without forcing a re-review of the control model.
- Matches how the two documents are already used in this repository — the tooling document already cross-references the architecture document's stage numbers rather than re-deriving them from scratch in several places.
- Introducing ADRs (ADR-0002) removes the single biggest current source of true duplication — the "Open issues" sections existed in near-identical form in both documents specifically because there was nowhere else to put a cross-cutting decision. Moving those to a shared `adr/` directory referenced from both documents removes that duplication without merging the documents themselves.
- Matches common practice for this kind of layered documentation (a stable "what and why" layer plus a separate, faster-moving "how" layer — the same split ADRs themselves formalize for point-in-time decisions).

**Cons**
- Still two files to keep in sync for anything that isn't cleanly separable into "control" vs. "implementation" — stage-by-stage prose descriptions, in particular, tend to want to exist in both.
- Requires discipline to actually cross-reference instead of re-describing; without that discipline this option degrades back into Option A's duplication risk with none of Option A's single-file benefit.

### Option C: Split further — three or more documents by artifact path or stage group

**Pros**
- Could make each document smaller and more focused (e.g., one document per artifact path).

**Cons**
- Most controls (Stage 2 egress, Stage 3 quarantine, Stage 11b recheck) are shared across all three artifact paths, so a per-path split would either duplicate shared content three times or require constant cross-document jumping for exactly the stages that most need to read coherently as a whole. Not pursued further.

## Decision outcome

**Chosen option: B — keep two documents, and use ADRs (ADR-0002) to eliminate the specific duplication that was previously forced into both documents' "Open issues" sections.** The architecture/tooling split reflects a real audience and change-velocity difference that a merge would erase; the duplication risk it carries is real but is mostly concentrated in exactly the content (open decisions, cross-cutting rationale) that ADRs are designed to hold in one place instead. Stage-by-stage description duplication that ADRs don't address is an accepted, smaller residual cost — see Consequences.

## Consequences

- The architecture and tooling documents remain separate files with separate revision histories.
- Cross-cutting decisions move to `adr/` (see ADR-0002) and are referenced, not restated, from both documents.
- Where stage-level prose still needs to exist in both documents (a real residual duplication, not eliminated by this decision), future edits should update both in the same commit — this was already the working pattern through the review-response pass, and remains a manual discipline rather than a structural guarantee.
- If a future revision finds the duplication cost still too high despite ADR adoption, that's grounds to revisit this ADR with a new one — not to silently drift the two documents apart.
