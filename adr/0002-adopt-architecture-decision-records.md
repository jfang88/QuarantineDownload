# ADR-0002: Adopt Architecture Decision Records in place of inline open issues

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-08-02 |
| Deciders | Security Architecture |
| Related | ADR-0001, all ADR-0004 and later |

## Context

Both `package-intake-architecture.md` and `solution-architecture-tooling.md` carried an "Open issues for internal review" section: free-text numbered questions with lettered options (A/B/C), embedded directly in the normative document. This worked as a first pass but had three problems, surfaced directly by the independent review and by maintaining it across two documents in practice:

1. It conflated "what the architecture currently says" with "what is still undecided," forcing a reader to figure out which parts of the document were settled and which were open questions dressed up in prose.
2. It left no clean record of *which* option was chosen once a decision was actually made — closing out an open issue meant either deleting the question (losing the rationale for why the other options were rejected) or leaving stale option text next to a decision, with no structural distinction between the two.
3. It existed in near-duplicate form in both documents (see ADR-0001), because there was no shared place to record a decision that affects both the control model and its implementation.

## Decision drivers

- Need a durable, dated record of *why* a decision was made, not just *what* was decided — useful when someone revisits a decision months later and wants to know if the circumstances that justified it still hold.
- Need one canonical location for a cross-cutting decision, referenced from both documents rather than duplicated in both.
- Should not require new tooling or process overhead disproportionate to the size of this repository.

## Considered options

### Option A: Keep the inline "Open issues" sections as-is

**Pros**
- No migration work.
- Reader doesn't need to leave the document to see open questions.

**Cons**
- Does not solve any of the three problems in Context — this is the status quo being replaced, not a genuine alternative.

### Option B: Adopt a lightweight ADR directory (`adr/`), one file per decision

**Pros**
- Standard, widely recognized pattern (MADR and similar templates) — low process overhead, just markdown files in a directory, no new tooling dependency.
- Each decision gets a stable identity (ADR number) that can be referenced from anywhere — commit messages, code comments, other ADRs, both main documents.
- Natural fit with "supersede, don't edit" semantics: when a decision changes, a new ADR replaces the old one and the old one's history stays intact, which the inline sections could not represent at all.
- Directly removes the duplication ADR-0001 identifies as the two-document split's main residual cost.

**Cons**
- One more directory to keep organized; requires the discipline described in `adr/README.md` (number sequentially, update status, cross-reference) to stay useful rather than becoming another form of drift.
- A reader who wants "the whole picture in one document" now has to open a second location for undecided questions — mitigated by keeping a short pointer section in both main documents rather than removing all trace of open questions from them.

### Option C: Adopt a heavier decision-record process (e.g., a formal RFC process with review SLAs, a decision-tracking ticket system integrated with GitLab issues)

**Pros**
- Would give firmer accountability (assigned reviewers, deadlines) than a markdown file.

**Cons**
- Disproportionate process overhead for a repository at this stage — no team, ticketing system, or review cadence currently exists to hang an RFC process off of. Revisit if and when this architecture moves from reference design to an actively staffed implementation program.

## Decision outcome

**Chosen option: B — adopt a lightweight `adr/` directory.** It solves all three problems in Context at a process cost proportional to this repository's current size and maturity, and is easy to upgrade to Option C later if the program grows into needing it.

## Consequences

- Both main documents' "Open issues for internal review" sections are replaced with a short section pointing to `adr/README.md`.
- Every item previously listed as an open issue becomes its own ADR file (ADR-0004 through ADR-0010), preserving the original pros/cons content rather than discarding it.
- Future cross-cutting decisions (not just the ones inherited from the original open-issues sections) should be recorded as new ADRs rather than added back into either main document as inline prose.
- This ADR itself is an example of the pattern: it documents a decision that is already `Accepted` because implementing it was the explicit request that produced this file.
