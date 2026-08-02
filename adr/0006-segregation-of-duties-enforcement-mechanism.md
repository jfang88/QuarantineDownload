# ADR-0006: Segregation-of-duties enforcement mechanism for pipeline administration

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-08-02 |
| Deciders | Security Architecture |
| Related | `package-intake-architecture.md` § Control objectives (segregation of duties), § Stage 11b administrative access; `solution-architecture-tooling.md` § Segregation of duties for the recheck job itself; ADR-0001 (GitLab CE licensing gap) |

## Context

The architecture requires that Nexus tag editing, CMDB trust-register edits, YARA ruleset promotion, and recheck-job configuration be restricted to a role distinct from artifact approval — but does not yet specify the technical enforcement mechanism, as opposed to documented policy. This is a narrower, tooling-focused sibling of the promotion-gate enforcement gap already resolved for GitLab CE (see the licensing/edition matrix): here the question is how *pipeline-admin* actions, not artifact promotions, get their separation of duties enforced.

## Decision drivers

- Documented-but-unenforced policy is not a control a security architecture should rely on for an audit trail.
- Different underlying tools (Nexus, the recheck-job repository, the CMDB) have different native RBAC granularity, which makes a uniform enforcement story harder than it looks.
- Enforcement mechanism should not require more new infrastructure than the risk being controlled justifies.

## Considered options

### Option A: RBAC roles enforced natively in each tool (Nexus roles, GitLab CODEOWNERS on the recheck-job repository, CMDB access groups), audited quarterly

**Pros**
- Uses each tool's actual access-control primitives rather than working around them — likely the most "correct" enforcement per tool.
- No new middleware to build.

**Cons**
- Enforcement granularity and audit-log quality varies by tool — a uniform SoD story requires stitching together several different audit trails rather than one.
- Quarterly audit cadence means a misconfigured native RBAC role could go unnoticed for up to a quarter.

### Option B: Route all pipeline-admin changes (ruleset promotion, trust register edits, recheck config) through a GitLab merge request requiring a second approver, giving a uniform audit trail across tools regardless of native RBAC granularity

**Pros**
- One consistent audit trail (Git history + MR approval record) regardless of which underlying tool the change ultimately affects.
- Simpler to reason about and to demonstrate to an auditor: "every pipeline-admin change is a reviewed MR" is a one-sentence control description.

**Cons**
- Only enforces separation for changes that actually go through Git-tracked configuration — a direct database edit to the CMDB trust register, for instance, would bypass it entirely. This is a real residual gap, not a hypothetical one.
- Inherits the GitLab CE required-approval enforcement gap identified elsewhere (ADR context: GitLab CE does not natively block a merge on missing approval) unless paired with the same Premium/Ultimate-or-scripted-CI-gate fix already needed for the promotion gate.

### Option C: Defer full enforcement to a later hardening phase and accept policy-only separation (documented, not technically enforced) for the initial rollout, with a stated date to revisit

**Pros**
- Fastest to ship; no new access-control work blocks initial rollout.

**Cons**
- Is, definitionally, the state this decision exists to move away from — "documented but unenforced" is explicitly called out in Context as insufficient. Choosing this option should be a deliberate, time-boxed risk acceptance, not a default.

## Decision outcome

*Undecided — pending review meeting sign-off.* Option B is the most likely candidate given it directly reuses infrastructure already being built for the promotion-gate fix, but it must be paired with a decision on the direct-database-edit gap (Option A for CMDB/Nexus specifically, layered on top of B for Git-tracked changes) rather than accepted as fully sufficient on its own.

## Consequences

*To be completed once a decision is recorded.* At minimum: whichever option is chosen should be implemented using the same enforcement pattern chosen for the Stage 9 promotion gate (see the licensing/edition matrix), not a second, differently-designed mechanism.

## Sources

The GitLab CE approval-enforcement gap this ADR's Option B works around was independently verified against [GitLab Docs — Merge request approvals](https://docs.gitlab.com/user/project/merge_requests/approvals/) — see `package-intake-architecture.md`'s References section for the verification record.
