# Architecture and Tooling Review Recommendations

## Review metadata

| Field | Value |
|---|---|
| Repository | `jfang88/QuarantineDownload` |
| Documents reviewed | `package-intake-architecture.md`, `solution-architecture-tooling.md`, `README.md` |
| Review date | 2026-08-02 |
| Review status | Living document — tracks resolution status against the two main documents as they change |
| Change type | Documentation and design corrections; no production implementation changes |
| Version | 1.1 |

### Revision history

| Version | Date | Summary of changes |
|---|---|---|
| 1.0 | 2026-08-02 | Initial review — 5 Critical, 7 High, 6 Medium findings, plus per-document edit lists, implementation sequence, acceptance criteria, and references |
| 1.1 | 2026-08-02 | Audited every finding against the current state of both main documents (verified by direct grep/read, not from memory). Added a resolution-status summary and a prominent "Outstanding" section grouping every not-fully-resolved item, so this document leads with what's still open rather than requiring a read-through to find it. Moved fully-resolved findings to an audit-trail section. Annotated the per-document edit lists, implementation sequence, and acceptance criteria with status. Added verification status to the references list. |

## Resolution status summary

| Category | Total | Fully resolved | Partially resolved (remnant open — see below) | Open |
|---|---:|---:|---:|---:|
| Critical (C1–C5) | 5 | 2 | 3 | 0 |
| High (H1–H7) | 7 | 3 | 4 | 0 |
| Medium (M1–M6) | 6 | 3 | 1 | 2 |
| **Total findings** | **18** | **8** | **8** | **2** |

"Partially resolved" means the core recommendation was implemented but at least one explicit sub-item from the original recommended change was not — these are not rounding errors, each remnant is named specifically in "Outstanding" below. Every status below was checked directly against the current file content on 2026-08-02, not asserted from memory of earlier work.

## Executive summary

The documents provide a strong foundation: they use a single controlled intake path, distinguish open-source packages, proprietary binaries, and AI/ML models, preserve chain of custody, and include post-approval re-evaluation and recall. The original review's largest concerns — overstating what a tool or data source proves, describing features that require different licences, and implementation examples that didn't match supported product APIs — have been substantially addressed. What remains open is narrower and mostly structural: a control-ID traceability matrix that was explicitly deferred rather than built, a handful of specific sub-recommendations that didn't get carried through when the parent finding was fixed (digest pinning, a storage-integrity hash recheck, basic-auth removal), and one deliberate product decision that went the opposite direction from this review's original recommendation (keeping the VirusTotal Public API in place, with caveats, rather than removing it).

## Outstanding — needs further work

Grouped by theme rather than by original finding ID, since several open items are remnants of findings that are mostly resolved. Each item states exactly what's missing and where it would go, not just which finding it traces to.

### 1. Documentation structure and traceability

- **No control-ID taxonomy or traceability matrix exists** (originally M1). `package-intake-architecture.md` still mixes principles, examples, and requirements in prose rather than stable control IDs (`INTAKE-*`, `FETCH-*`, etc.) with MUST/SHOULD/MAY language mapped to threat/owner/evidence/test. This was evaluated and explicitly deferred — see [ADR-0010](adr/0010-control-id-taxonomy-and-traceability-matrix.md) (`Proposed`), which records the reasoning (large authoring effort, only worth it if the document needs to serve an external audit function) rather than silently dropping the recommendation.
- **No non-functional requirements section exists** (originally M4). Expected artifact volume, peak concurrency, largest model/firmware size, scan/approval SLA, data residency, retention, availability target, GPU/test-hardware needs, and external-service-outage behaviour are not recorded anywhere in either main document. [ADR-0011](adr/0011-backup-rpo-rto-targets.md) covers backup RPO/RTO specifically (one sub-piece of this), and the POC documents size for a demo environment, but the production-scale NFRs this finding asked for don't exist yet. Nothing currently tracks this as a decision or a to-do beyond this entry — it needs either a new ADR or a dedicated section, not further deferral by omission.
- **The "Prompt to recreate this document" section was never moved or removed** (part of M5). It's still present verbatim at the bottom of `package-intake-architecture.md`. The original recommendation was to move it to a contributor/development note or drop it from the controlled architecture record, on the reasoning that a reproducibility prompt reads oddly inside a document meant to carry governance authority.
- **Hard-coded version claims in prose were never replaced with a maintained BOM** (part of M5). Tool versions (e.g. `dependencytrack/apiserver:5.6.0`, `v1.18.0` for Syft) are still embedded directly in prose and code examples throughout `solution-architecture-tooling.md` rather than referencing a separately maintained, single-source tool bill of materials. Every version bump currently means hunting through prose for every mention.
- **"Binary recheck" was never renamed to "artifact re-evaluation"** (recommended-edit item 1 for `package-intake-architecture.md`). The document still uses "binary recheck" (9 occurrences, including the Stage 11b heading) despite Stage 11b covering packages, firmware, and models, not just binaries. Purely a terminology fix, never actioned.

### 2. Pipeline trust-root and workload identity hardening

- **Basic-auth is still used in Nexus API examples** (remnant of H6). The `curl | sh` piping H6 specifically called out was fixed (see Resolved, below), but the broader "remove short-lived-credential violations from examples" intent wasn't fully carried through — `curl -u user:pass` still appears 3 times in `solution-architecture-tooling.md`'s Nexus tagging API examples.
- **No workload identity (OIDC/mTLS) section exists** (recommended-edit item 9 for `solution-architecture-tooling.md`, and part of H6). Neither main document discusses short-lived workload credentials, key rotation, or trust-store administration for the pipeline's own service accounts — everything currently assumes static credentials (env vars, `.env` files) with no rotation story beyond "don't commit them."
- **No tested tool bill of materials, compatibility matrix, or rollback procedure exists** (recommended-edit item 10 for `solution-architecture-tooling.md`, and part of H6). The pipeline-tooling-supply-chain section covers pinning and signature verification for individual tool installs, but there's no consolidated "here is the tested combination of versions, and here is how to roll back a bad upgrade" artifact.

### 3. Evidence completeness and integrity

- **No scheduled storage-integrity recheck exists** (remnant of H2). Stage 11b rechecks stored hashes against *external* threat intelligence (VirusTotal, MalwareBazaar, YARA) on a schedule, but nothing recomputes the actual stored bytes' hash and compares it against the original intake-time value in an append-only ledger — i.e., nothing currently catches storage-layer corruption or tampering that doesn't happen to also trip an external reputation check. This is a distinct control from the recheck logic that does exist and should be added alongside it, not folded into it.
- **The partial-BOM evidence fields are missing one of five requested fields** (remnant of H3). Both main documents record `coverage`, `generator`, `source`, and `confidence` for partial/vendor BOMs, but not a distinct `completeness` field — a minor gap, but a specifically named one in the original recommendation.
- **Dependency-Track container images are pinned by tag, not digest** (remnant of H5). The Stage 5a deployment example uses `dependencytrack/apiserver:5.6.0` / `dependencytrack/frontend:5.6.0` — a tag can be force-moved by the vendor (rare, but possible) in a way a digest pin (`@sha256:...`) cannot. The rest of the pipeline-tooling-supply-chain section does require digest pinning for scanner tools; this specific example wasn't brought in line with that same standard.

### 4. Cosmetic/labeling cleanup

- **"Complete recommended self-hosted free stack" was never renamed** (remnant of C4). The original recommendation was to rename this section to "Recommended stack and licence assumptions" and add a Free/Pro/Premium/commercial bill of materials. The Licensing and edition decision matrix section added elsewhere in `solution-architecture-tooling.md` covers the licensing bill-of-materials intent, but the specific section this finding pointed at still carries its original, licence-blind title.
- **Nexus metadata examples remain illustrative rather than Swagger-generated** (remnant of C5). The examples are now explicitly captioned "illustrative — validate against the deployed instance's Swagger spec before implementing" rather than presented as ready-to-use, which resolves the misleading-confidence problem the finding raised. They were not literally regenerated from a live Nexus Swagger specification, which isn't fully achievable in a documentation-only repository without a running Nexus instance to generate against — flagged here so that constraint is explicit rather than silently accepted.

### 5. A deliberate decision that went against this review's original recommendation

- **C2's specific recommendation — remove the VirusTotal Public API from the baseline stack — was not followed.** By explicit product direction, VirusTotal was kept in the baseline with added caveats (growth-over-time framing, ToS risk flagged, a note that a commercial licence is likely needed eventually) instead of being removed. This is a legitimate outcome, not an oversight, but it means the underlying licensing/procurement question C2 raised is still open — it's now tracked as [ADR-0004](adr/0004-virustotal-public-api-usage-and-licensing.md) (`Proposed`), which lays out the same options C2's recommended change did (remove, cap at Tier 1, replace with a commercial feed, or get a legal read on current usage) for a review meeting to actually decide among.

## Resolved findings

These were checked directly against the current documents, not assumed. Each entry says where the fix landed.

### Critical

- **C1 (GitLab CE approval enforcement)** — Resolved. `solution-architecture-tooling.md` now has a Licensing and edition decision matrix stating the CE/Premium boundary explicitly, plus a scripted-CI-status-check alternative in Stage 9 for organizations staying on CE.
- **C3 (NSRL format/semantics)** — Resolved. Both documents now describe RDSv3 SQLite distribution, record `present_in_nsrl`/`not_present`/`dataset_version`, and the Stage 11b "known-bad status changed" action was removed and replaced with a note that NSRL has no malicious/benign verdict to change.

### High

- **H1 (dynamic analysis must be artifact-specific)** — Resolved. Both documents now define per-platform analysis profiles (Windows user mode / kernel, Linux, firmware, containers, models, non-executable data) with PASS/FAIL/INCONCLUSIVE/UNAVAILABLE outcomes and fail-closed rules for the latter two, replacing the blanket "mandatory CAPE for everything" requirement.
- **H4 (Microsoft update distribution)** — Resolved. The WSUS/Update Catalog language was corrected to the real UpdateID-import workflow; the fictitious "Microsoft Update Catalog API" was replaced with the MSRC CVRF/CSAF API plus the documented WSUS import path.
- **H7 (signature/revocation nuance)** — Resolved. Linux verification now distinguishes RPM/APT repository-metadata trust from optional per-package `dpkg-sig` signing; Authenticode revocation handling now checks trusted timestamp and revocation reason before auto-blocking, rather than blocking on a bare OCSP "revoked" response.

### Medium

- **M2 (formal lifecycle state machine)** — Resolved. `package-intake-architecture.md` now has a 19-state lifecycle model, a `stateDiagram-v2`, a transitions/actors/evidence table, and idempotency/reconciliation rules, adopted via [ADR-0003](adr/0003-adopt-artifact-lifecycle-state-machine.md) (`Accepted`). Nexus repository groups are explicitly described as a reconciled reflection of this canonical state, not the state machine itself.
- **M3 (hardened downloader requirements)** — Resolved. Stage 2 in both documents now requires redirect revalidation, RFC1918/loopback/link-local/metadata-service blocking, DNS-rebinding defenses, and acquisition evidence capture (DNS, TLS, redirect chain, headers).
- **M6 (repository entry point)** — Resolved. `README.md` has a title, the "architeture" typo is fixed, it links to every document in the repository including this one and the ADR register, and it states draft/reference status explicitly rather than implying a deployed system.

### Findings resolved except for a specific named remnant

These four Critical/High findings and one Medium finding had their core recommendation implemented, but each has at least one specific sub-item still open — see "Outstanding" above for the exact remaining piece, referenced by finding ID:

- **C4** (Repository Firewall vs. intake repository) — the three-concept split and the evidence database were built; the specific section rename was not (Outstanding § 4).
- **C5** (Nexus metadata API examples) — the evidence-database correction and the "illustrative, validate against Swagger" caveat were added; literal Swagger-generated examples were not produced (Outstanding § 4).
- **H2** (model ML-BOM/resource controls) — ML-BOM, the mutable-source-reference-drift rename, and resource-abuse limits were all built; the scheduled storage-integrity hash recheck against an append-only ledger was not (Outstanding § 3).
- **H5** (Dependency-Track deployment) — the pinned 5.x/PostgreSQL architecture and the where-used-versus-deployment-inventory correction were both made; digest pinning for the DT images specifically was not carried through (Outstanding § 3).
- **H6** (trust-root policy contradiction) — the `curl | sh` piping was fixed with a download-verify-install pattern; basic-auth removal, workload identity, and a tested tool BOM/rollback procedure were not addressed (Outstanding §§ 1–2).
- **M5** (separate normative architecture from implementation) — Architecture Decision Records were adopted, which was this finding's main ask; the recreate-prompt relocation and the hard-coded-version-to-maintained-BOM change were not done (Outstanding § 1).

## Recommended edits by document

Status legend: ✅ done · 🟡 partial (remnant in Outstanding above) · ❌ not started.

### `package-intake-architecture.md`

1. ❌ Change "binary recheck" to **artifact re-evaluation** throughout.
2. ✅ Correct the VirusTotal, NSRL and model-reference semantics.
3. ✅ Replace the single Stage 6 sandbox with artifact-specific analysis profiles.
4. ✅ Add ML-BOM and partial/vendor BOM handling (🟡 `completeness` field still missing — see Outstanding § 3).
5. ✅ Add the lifecycle state machine, scanner outcome states and fail-closed rules.
6. ✅ Separate Microsoft patch governance from the general Nexus distribution path.
7. ✅ Add secure-fetch controls and immutable acquisition evidence.
8. ❌ Add control IDs and a control-to-evidence traceability table (deferred via ADR-0010).
9. ❌ Remove or relocate the document recreation prompt.

### `solution-architecture-tooling.md`

1. 🟡 Add a licensing/edition matrix and correct GitLab CE, Nexus Pro and Repository Firewall assumptions (matrix added; the "Complete recommended self-hosted free stack" section itself was never renamed — Outstanding § 4).
2. 🟡 Replace unsupported Nexus metadata API examples with validated Swagger-based examples and a separate evidence schema (evidence schema done; examples are captioned illustrative rather than literally Swagger-generated — Outstanding § 4).
3. 🟡 Replace the VirusTotal Public API baseline with a licensed/optional integration (kept in place by deliberate decision instead — Outstanding § 5, ADR-0004).
4. ✅ Update NSRL ingestion to RDSv3 SQLite or documented ETL.
5. 🟡 Replace the legacy Dependency-Track bundled deployment with a pinned 5.x architecture (done; images pinned by tag not digest — Outstanding § 3).
6. 🟡 Replace `curl \| sh` installation patterns with verified artifact installation (the install-script piping is fixed; basic-auth in API examples elsewhere was not addressed — Outstanding § 2).
7. ✅ Split CAPE, Linux, firmware and model analysis tooling by profile.
8. ✅ Replace the Microsoft Catalog "API" scripts with the documented WSUS UpdateID import workflow.
9. ❌ Add workload identity, secrets handling, key rotation and trust-store administration.
10. ❌ Add a tested tool BOM, compatibility matrix and upgrade/rollback procedure.

### `README.md`

1. ✅ Correct the spelling error and add a document index.
2. ✅ State that the repository contains a draft/reference architecture, not a deployed implementation.
3. ✅ Summarise the three intake paths and cross-cutting controls in a shorter form, leaving detail in the linked documents.

## Proposed implementation sequence

Status reflects design/documentation work only — none of these steps have been executed as an actual running system; this repository remains documentation and a POC *plan*, not a deployed environment.

1. ✅ **Correct factual and licensing assumptions** — C1 through C5 (with the specific remnants noted above).
2. 🟡 **Approve the target control model** — lifecycle states (✅), evidence schema (✅), analysis profiles (✅), and Microsoft patch boundary (✅) are all designed; "approve" implies a sign-off step that hasn't formally happened outside of this repository's own ADR process.
3. ❌ **Resolve architecture decisions** — GitLab tier/gating (ADR-0006, open), repository platform/licences (ADR-0012, open), threat-intelligence provider (ADR-0004, open), evidence database (designed but not an ADR — arguably should be, since Stage 3's evidence-database concept was never itself formalized as a decision record), model registry (ADR-0007, open), deployment inventory (not tracked as a distinct decision). Nine open ADRs in total — see `adr/README.md`.
4. ✅ **Build a thin proof of concept** — done at the *design* level: `poc-deployment-plan.md` and `poc-build-runbook.md` describe exactly this (one package ecosystem, one generic binary type, immutable acquisition evidence, no dynamic execution in the core profile). Not yet built as running code.
5. ✅ **Add dynamic analysis profiles one platform at a time** — designed (per-platform profiles exist in both main documents and the POC plan describes phased rollout), not yet implemented as running code.
6. ❌ **Implement re-evaluation and recall** — designed in detail (Stage 11b, the lifecycle state machine's `SUSPENDED`/`RECALLED` states), but explicitly gated on "reliable deployment inventory and block-new-consumption controls," neither of which exist yet since nothing has been built.
7. ✅ **Add model intake** — done at the design level (Path C, ML-BOM, resource controls, safe-format conversion policy all exist in both main documents).
8. ❌ **Run abuse, recovery and recall exercises** — not started; no implementation exists yet to run exercises against.

## Acceptance criteria for the next document revision

Re-evaluated against the current documents:

- 🟡 No mandatory control depends on an optional feature of the selected licence tier — true for the promotion gate (C1 fixed with an explicit CE-vs-Premium choice); **not yet true for VirusTotal**, which remains a mandatory-by-default control sourced from a licence tier (Public API) whose terms may not permit the use this architecture makes of it (see Outstanding § 5).
- ✅ Every external API is approved for the intended enterprise use and data-sharing model — the WSUS/Update Catalog and Nexus API corrections address the two APIs that were previously described inaccurately; the VirusTotal ToS question is tracked (not silently resolved) via ADR-0004.
- ✅ Every evidence source documents what it proves and what it does not prove — NSRL, VirusTotal, MalwareBazaar, and Dependency-Track where-used all now carry explicit "this proves X, not Y" language.
- ✅ Every artifact type has a supported analysis profile or an explicit unsupported outcome — the Stage 6 analysis-profiles table plus the PASS/FAIL/INCONCLUSIVE/UNAVAILABLE outcome states satisfy this.
- 🟡 The promotion gate can be tested automatically and cannot be bypassed by the requestor — true as *designed* (the scripted CI-status-check alternative for GitLab CE is specified in enough detail to build); not yet true in practice since nothing has been implemented or tested.
- 🟡 Repository APIs and examples are validated against the deployed product's Swagger/API specification — the caveat directing implementers to do this was added; the examples themselves were not regenerated from an actual Swagger spec (Outstanding § 4).
- ✅ Artifact bytes, metadata and approval evidence are linked by an immutable hash and auditable state transition — the lifecycle state machine's append-only transition log design satisfies this at the design level.
- 🟡 Recall can prevent new consumption, preserve evidence, identify deployed instances and track remediation — designed (Stage 10/11b push-remediation, the `SUSPENDED`/`RECALLED` states); "identify deployed instances" still depends on the deployment-inventory work flagged as unresolved in the implementation sequence above.
- ❌ Backup restore, scanner outage, external-intelligence outage and false-positive scenarios are documented and tested — documented (backup/recovery section, `fp_suppressions` table, INCONCLUSIVE/UNAVAILABLE handling), but "tested" requires a running implementation, which doesn't exist yet. The POC's acceptance criteria (POC-13, POC-14 in `poc-deployment-plan.md`) cover some of this once the POC is actually built.

## Authoritative references consulted

Verification status as of the repository-wide references review (2026-08-02): "Re-verified" means independently checked against the live source again during that pass (see `package-intake-architecture.md` and `solution-architecture-tooling.md`'s own References sections for the verification records); "as originally cited" means not re-fetched, but no reason found to doubt it. All ten remain relevant — each still backs a claim that is currently live in one of the two main documents, so none should be pruned.

| Reference | Status |
|---|---|
| GitLab Docs, **Merge request approvals**: https://docs.gitlab.com/user/project/merge_requests/approvals/ | Re-verified |
| VirusTotal Docs, **Public vs Premium API**: https://docs.virustotal.com/reference/public-vs-premium-api | Re-verified |
| NIST NSRL, **Current RDS Hash Sets**: https://www.nist.gov/itl/csd/secure-systems-and-applications/national-software-reference-library-nsrl/nsrl-download-0 | Re-verified |
| Sonatype, **Repository Firewall** and **Firewall Quarantine**: https://help.sonatype.com/en/repository-firewall.html and https://help.sonatype.com/en/firewall-quarantine.html | As originally cited (not re-fetched) |
| Sonatype, **Tagging**: https://help.sonatype.com/en/tagging.html | As originally cited (not re-fetched) |
| Dependency-Track releases: https://github.com/DependencyTrack/dependency-track/releases | Re-verified |
| CycloneDX, **Machine Learning Bill of Materials**: https://cyclonedx.org/capabilities/mlbom/ | Re-verified |
| PyTorch, **torch.load security warning**: https://docs.pytorch.org/docs/stable/generated/torch.load.html | Re-verified (via PyTorch's security policy page, not this exact URL — see `package-intake-architecture.md`'s References for the specific source used) |
| Microsoft Learn, **WSUS and the Microsoft Update Catalog**: https://learn.microsoft.com/en-us/windows-server/administration/windows-server-update-services/manage/wsus-and-the-catalog-site | Re-verified |
| Aqua Security / GitHub advisory, **Trivy ecosystem supply chain temporarily compromised**: https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23 | Re-verified |
