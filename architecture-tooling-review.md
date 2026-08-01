# Architecture and Tooling Review Recommendations

## Review metadata

| Field | Value |
|---|---|
| Repository | `jfang88/QuarantineDownload` |
| Documents reviewed | `package-intake-architecture.md`, `solution-architecture-tooling.md`, `README.md` |
| Review date | 2026-08-02 |
| Review status | Recommendations for architecture review |
| Change type | Documentation and design corrections; no production implementation changes |

## Executive summary

The documents provide a strong foundation: they use a single controlled intake path, distinguish open-source packages, proprietary binaries, and AI/ML models, preserve chain of custody, and include post-approval re-evaluation and recall. The largest remaining risks are not missing security products; they are places where the documents overstate what a tool or data source proves, describe features that require different licences, or provide implementation examples that do not match supported product APIs.

Before implementation begins, revise the design so that:

1. Every mandatory approval control has an enforceable implementation on the selected product tier.
2. External threat-intelligence services are used under terms that permit enterprise automation.
3. Reference datasets and scanner results are treated as evidence with confidence and provenance, not as proof that an artifact is safe.
4. Repository Firewall quarantine, the enterprise intake repository, and promotion state are modeled as separate capabilities.
5. Analysis is selected by artifact format and execution environment rather than sending every proprietary artifact to a Windows malware sandbox.
6. Models receive a machine-readable ML bill of materials and resource-abuse controls in addition to safe-serialization rules.
7. All examples use supported APIs, workload identity, immutable references, and verified installation artifacts.

## Priority findings

### Critical — correct before selecting or purchasing tooling

#### C1. GitLab CE does not enforce the approval gate described in the documents

The tooling guide recommends GitLab Community Edition and then relies on required merge-request approvals for promotion and segregation of duties. GitLab Free permits approvals, but they are optional and do not prevent merging without approval. Required approval rules and stronger approval settings are Premium/Ultimate features.

**Recommended change**

Choose one enforceable pattern and state it explicitly:

- License GitLab Premium/Ultimate and configure required approvals, protected branches, prevention of author approval, and approval reset after new commits; or
- Keep GitLab CE, but make a protected-branch CI status check the promotion gate. The check must verify two independent signed approvals from an external policy service or approval database, and only a restricted service account may merge after the check passes.

Do not describe native GitLab CE merge-request approvals as a production control.

#### C2. The VirusTotal Public API cannot be the baseline enterprise recheck service

The documents treat 500 lookups/day as the main constraint. VirusTotal also states that its Public API must not be used in commercial products, services, or business workflows. An automated enterprise intake and nightly recheck workflow therefore requires a commercial agreement/Premium API or a different licensed intelligence source.

**Recommended change**

- Remove the Public API from the recommended baseline stack.
- Make VirusTotal Premium, ReversingLabs, Recorded Future, or another appropriately licensed service an optional integration selected during procurement.
- Retain an offline-first baseline using local YARA, ClamAV, vendor advisories, signed provenance, and an approved local reputation dataset.
- Add a privacy review: even hash-only queries can disclose what software or models the organisation is evaluating or using.
- Replace the fixed “two engines” rule with a weighted policy using engine quality, result freshness, prevalence, corroborating evidence, and analyst review.

#### C3. NSRL is described with the wrong data format and overly strong semantics

The current NIST Reference Data Set is distributed as RDSv3 SQLite. The documents instead prescribe a PostgreSQL table and describe NSRL as a “known-good” or possible “known-bad” status source. NSRL is a reference corpus. A match supports identification and provenance, but does not prove that a file is currently safe, authorised, or uncompromised. The architecture’s Stage 11b “known-bad status changed” logic should be removed.

**Recommended change**

- Ingest RDSv3 SQLite directly, or implement a documented ETL process into a separately managed database.
- Record results as `present_in_nsrl`, `not_present`, `dataset_version`, and matched product metadata.
- Never use absence as a negative signal or presence as an approval bypass.
- Remove the “NSRL changed to known-bad” action. Use malware intelligence, revocation data, vendor recalls, and incident intelligence for negative assertions.

#### C4. Sonatype Repository Firewall quarantine is conflated with the custom intake repository

Repository Firewall is a separately licensed capability powered by IQ Server and quarantines components requested through proxy repositories. It is not the same as a generic immutable quarantine/hosted repository for arbitrary installers, firmware, and models. The documents also label a stack containing Nexus Repository Pro and Repository Firewall as “self-hosted free.”

**Recommended change**

Model three separate concepts:

1. **Ecosystem proxy control** — proxy repositories protected by Repository Firewall or an equivalent policy engine.
2. **Enterprise intake evidence store** — a hosted/raw repository or object store containing the exact downloaded bytes, immutable hashes, reports, and attestations.
3. **Approved distribution repositories** — format-specific hosted repositories used by consumers.

Rename “Complete recommended self-hosted free stack” to “Recommended stack and licence assumptions” and add a bill of materials showing Free, Pro, Premium, and optional commercial dependencies.

#### C5. The Nexus metadata examples do not match the supported tagging model

The guide shows a `PUT /service/rest/v1/components/{id}/tags` request containing arbitrary key/value fields. Nexus tags are created through the Tags API and applied at component level, not individual asset level. Tags may contain custom JSON attributes, but they are not a substitute for a strongly typed per-artifact evidence database, particularly when one component can contain multiple assets.

**Recommended change**

- Replace pseudo-API examples with calls generated from the installed Nexus Swagger specification.
- Use a separate evidence database keyed by repository, format, component coordinates, asset path, and SHA-256.
- Store only compact lookup identifiers and lifecycle tags in Nexus.
- Attach immutable JSON evidence bundles and attestations to the artifact or evidence store.
- Define canonical metadata schemas and validation, rather than adding ad hoc tag names in pipeline scripts.

### High — correct before detailed implementation

#### H1. Dynamic analysis must be artifact-specific

“Mandatory CAPE detonation for all proprietary binaries, drivers, and firmware” is not implementable as written. CAPE is primarily a Windows malware-analysis sandbox. A Windows guest cannot meaningfully execute Linux packages, arbitrary firmware, device drivers, or large model files.

**Recommended change**

Define analysis profiles selected by detected format and declared platform:

| Profile | Typical artifacts | Dynamic or specialised analysis |
|---|---|---|
| Windows user mode | EXE, MSI, DLL, scripts | CAPE or isolated Windows VM |
| Windows kernel | Drivers | Signature and catalog checks, static driver analysis, isolated test VM/hardware where supported |
| Linux | ELF, RPM, DEB, shell installers | Disposable Linux VM/Kata/gVisor, eBPF/syscall and network monitoring |
| Firmware | Images, update capsules | Firmware extraction, file-system analysis, binwalk-style inspection, vendor signature checks, QEMU/FirmAE or test hardware where feasible |
| Containers | OCI images | Layer/SBOM analysis plus sandboxed runtime tests |
| Models | Safetensors, GGUF, legacy checkpoints | Resource-limited parser/load harness with no credentials or egress |
| Non-executable data | Archives, datasets | Structural validation, content policy, decompression-bomb controls |

Each profile must return `PASS`, `FAIL`, `INCONCLUSIVE`, or `UNAVAILABLE`. Define fail-closed and exception behaviour for the latter two states.

#### H2. Model safety and provenance need an ML-BOM, immutable-hash recheck, and resource controls

The documents correctly prefer `safetensors` and GGUF and warn about pickle. However, “no SBOM applies” and “no code execution surface” are too absolute. CycloneDX supports an ML-BOM and model-card data. Data-only formats reduce deserialization risk but parsers can still have vulnerabilities, and malicious files can cause memory, disk, CPU, or decompression exhaustion.

**Recommended change**

- Generate a CycloneDX ML-BOM/model record for every approved model, including files, hashes, source namespace, immutable revision, license, datasets, base-model lineage, framework/runtime requirements, and model-card limitations.
- Treat the hub commit SHA as immutable. A branch or tag can move; the pinned commit does not “silently repoint.” Rename this check to **mutable-source-reference drift**.
- Recompute the stored Nexus/object-store hash on a schedule and compare it to an append-only intake ledger.
- Enforce limits for total bytes, tensor dimensions, declared allocation, RAM, CPU/GPU time, temporary disk, archive depth, and compression ratio.
- Default-deny pickle-based checkpoints. An exception should use a disposable VM or strong sandbox, no secrets, strict resource limits, and conversion to a safe format followed by re-ingestion. Static pickle scanners are supporting evidence, not proof of safety.

#### H3. Closed-source artifacts should not be treated as categorically incapable of BOM data

A proprietary binary may have a vendor-provided SBOM, embedded package metadata, or a partial component inventory. The correct distinction is completeness and evidence quality, not open versus closed source.

**Recommended change**

- Accept and verify vendor SBOMs where provided.
- Generate a partial binary/component inventory where technically possible.
- Record `coverage`, `generator`, `source`, `confidence`, and `completeness` fields.
- Do not upload an empty BOM or present partial data as complete.
- Continue vendor advisory monitoring as a complementary control, not a replacement for all BOM data.

#### H4. Microsoft update distribution should remain on supported Microsoft channels

The guide proposes configuring WSUS/SCCM/Intune to “pull from Nexus.” That is not the standard supported Microsoft patch flow. Microsoft documents importing Update IDs from the Update Catalog into WSUS, after which WSUS manages metadata, content download, approval, and deployment.

**Recommended change**

- Keep WSUS/Configuration Manager/Intune as the authoritative distribution and compliance channel for Microsoft updates.
- Use the intake workflow to approve the Update ID, capture catalog metadata, independently verify downloaded evidence where required, and authorise import/approval in WSUS.
- Store a copy and evidence in Nexus only when policy requires archival; do not imply that Nexus replaces WSUS content distribution without a separately validated supported design.
- Replace “Microsoft Update Catalog API” with the supported UpdateID/WSUS import workflow or a documented vendor interface. Do not rely on scraping.

#### H5. Dependency-Track deployment instructions and inventory claims need revision

The quick-start uses the legacy bundled container and an unpinned image. Dependency-Track 5 is container-only, requires PostgreSQL 14+, and has explicit migration requirements. Also, “where used” represents projects and BOM relationships; it is not proof that a component is installed on a runtime host.

**Recommended change**

- Pin a tested Dependency-Track 5.x deployment including API server, frontend, PostgreSQL, health checks, backups, and migration procedure.
- Pin images by digest and route upgrades through the same tool-intake control.
- Define separate inventories for build dependency relationships, release manifests, runtime deployment, and endpoint software inventory.
- Use CMDB/EDR/container orchestration/deployment telemetry for actual deployed-instance recall.

#### H6. The trust-root implementation contradicts its own policy

The guide says tooling must be provenance-verified, but its Dockerfile examples pipe remote installer scripts directly into `sh`. Version strings alone do not provide integrity and a mutable installation script can change independently.

**Recommended change**

- Download a release artifact and checksum/signature from independently authenticated locations.
- Verify signature, checksum, expected identity, and transparency-log entry before installation.
- Pin container images and GitHub Actions by immutable digest/commit SHA.
- Build scanner images through the same quarantine and approval process they enforce.
- Use short-lived OIDC/mTLS workload identity; remove basic-auth credentials from examples.
- Keep a tested scanner bill of materials and a rollback image.

The 2026 Trivy compromise is a useful case study for this control, but the architecture should avoid permanent product bans based on prose that will become stale. Selection should be based on a currently supported fixed version, independent verification, defence in depth, and operational testing.

#### H7. Signature and revocation policy needs platform-specific nuance

The Linux section presents `dpkg-sig` as a universal native DEB verification mechanism, but Debian-family trust commonly comes from signed repository metadata, while vendor packages vary. The Authenticode section also suggests auto-blocking based solely on a current OCSP result without considering trusted timestamps, revocation reason, or effective date.

**Recommended change**

- Define per-source verification: RPM package signatures, Debian `InRelease`/`Release.gpg` repository metadata, detached vendor signatures, Windows catalog/Authenticode, and firmware/vendor-specific signing.
- Store publisher identity as a managed trust policy supporting planned key/certificate rotation, not a single forever-thumbprint.
- For Authenticode, evaluate the signature, trusted timestamp, chain policy at signing time and evaluation time, revocation reason/effective date, and vendor incident notice before automated recall.
- Use “quarantine and deny new consumption” as the first automated response when evidence is high confidence; preserve the artifact for investigation.

### Medium — improve maintainability and auditability

#### M1. Add a normative control catalogue and traceability matrix

The documents mix principles, examples, product choices, and policy requirements. Introduce stable control IDs and RFC-style language (`MUST`, `SHOULD`, `MAY`). Map each control to threat, owner, implementation, evidence, failure mode, metric, and test.

Suggested control groups:

- `INTAKE-*` request, identity, source and approval
- `FETCH-*` downloader and egress safety
- `AUTH-*` integrity, signing and provenance
- `ANALYSIS-*` static/dynamic analysis
- `PROMOTE-*` evidence and promotion
- `CONSUME-*` internal-only resolution and pinning
- `MONITOR-*` continuous re-evaluation
- `RECALL-*` deny, preserve, identify and remediate
- `PLATFORM-*` pipeline trust, audit and recovery

#### M2. Add a formal artifact lifecycle state machine

Use explicit states such as:

`REQUESTED → APPROVED_TO_FETCH → ACQUIRED → ANALYSING → INCONCLUSIVE/REJECTED/COOLING → TESTED → APPROVED → SUSPENDED/RECALLED → EXPIRED`

Define who can perform each transition, required evidence, idempotency, retry behaviour, and how state is reconciled between the portal, evidence database, repository and CMDB. Repository groups should not be treated as the complete state machine.

#### M3. Add hardened downloader requirements

The fetch service is a high-risk server-side URL fetcher and needs controls beyond a domain allowlist:

- Revalidate every redirect and final destination.
- Block loopback, link-local, RFC1918/private, metadata-service and internal DNS targets.
- Defend against DNS rebinding and enforce approved resolved IP ranges where appropriate.
- Enforce TLS policy, hostname verification, timeouts and maximum redirects.
- Check file magic against declared type.
- Limit compressed and uncompressed size, archive depth, file count and compression ratio.
- Stream to immutable storage while calculating hashes; never execute from the download directory.
- Record DNS, TLS certificate, redirect chain, HTTP headers and source timestamp as acquisition evidence.

#### M4. Add explicit non-functional requirements

Before product selection, record expected artifact volume, peak concurrency, largest model/firmware size, scan and approval SLA, cooling-off exceptions, data residency, retention, RPO/RTO, availability target, GPU/test-hardware needs, and external-service outage behaviour. These requirements will determine whether the proposed free-first stack is viable.

#### M5. Separate normative architecture from implementation notes

- Keep `package-intake-architecture.md` product-neutral and control-focused.
- Keep concrete commands, product versions and API examples in `solution-architecture-tooling.md`.
- Move the “Prompt to recreate this document” sections to a contributor/development note or remove them from controlled architecture records.
- Add Architecture Decision Records for unresolved choices rather than duplicating open-issue text in both documents.
- Replace hard-coded version claims in prose with a tested implementation BOM maintained separately.

#### M6. Improve the repository entry point

Update `README.md` to:

- Add a title and correct “architeture.”
- Link to the architecture, tooling guide, this review, and future ADRs.
- State document status, intended audience and what is not implemented yet.
- Avoid presenting draft architecture as a deployed control environment.

## Recommended edits by document

### `package-intake-architecture.md`

1. Change “binary recheck” to **artifact re-evaluation** throughout, because the stage includes packages, firmware and models.
2. Correct the VirusTotal, NSRL and model-reference semantics described above.
3. Replace the single Stage 6 sandbox with artifact-specific analysis profiles.
4. Add ML-BOM and partial/vendor BOM handling.
5. Add the lifecycle state machine, scanner outcome states and fail-closed rules.
6. Separate Microsoft patch governance from the general Nexus distribution path.
7. Add secure-fetch controls and immutable acquisition evidence.
8. Add control IDs and a control-to-evidence traceability table.
9. Remove or relocate the document recreation prompt.

### `solution-architecture-tooling.md`

1. Add a licensing/edition matrix and correct GitLab CE, Nexus Pro and Repository Firewall assumptions.
2. Replace unsupported Nexus metadata API examples with validated Swagger-based examples and a separate evidence schema.
3. Replace the VirusTotal Public API baseline with a licensed/optional integration.
4. Update NSRL ingestion to RDSv3 SQLite or documented ETL.
5. Replace the legacy Dependency-Track bundled deployment with a pinned 5.x architecture.
6. Replace `curl | sh` installation patterns with verified artifact installation.
7. Split CAPE, Linux, firmware and model analysis tooling by profile.
8. Replace the Microsoft Catalog “API” scripts with the documented WSUS UpdateID import workflow.
9. Add workload identity, secrets handling, key rotation and trust-store administration.
10. Add a tested tool BOM, compatibility matrix and upgrade/rollback procedure.

### `README.md`

1. Correct the spelling error and add a document index.
2. State that the repository contains a draft/reference architecture, not a deployed implementation.
3. Summarise the three intake paths and cross-cutting controls in a shorter form, leaving detail in the linked documents.

## Proposed implementation sequence

1. **Correct factual and licensing assumptions** — C1 through C5.
2. **Approve the target control model** — lifecycle states, evidence schema, analysis profiles and Microsoft patch boundary.
3. **Resolve architecture decisions** — GitLab tier/gating, repository platform/licences, threat-intelligence provider, evidence database, model registry and deployment inventory.
4. **Build a thin proof of concept** for one package ecosystem and one generic binary type, with immutable acquisition evidence and no dynamic execution.
5. **Add dynamic analysis profiles** one platform at a time, with explicit unsupported/inconclusive outcomes.
6. **Implement re-evaluation and recall** only after reliable deployment inventory and block-new-consumption controls exist.
7. **Add model intake** with ML-BOM, resource controls and safe-format conversion.
8. **Run abuse, recovery and recall exercises** before declaring the design production-ready.

## Acceptance criteria for the next document revision

- No mandatory control depends on an optional feature of the selected licence tier.
- Every external API is approved for the intended enterprise use and data-sharing model.
- Every evidence source documents what it proves and what it does not prove.
- Every artifact type has a supported analysis profile or an explicit unsupported outcome.
- The promotion gate can be tested automatically and cannot be bypassed by the requestor.
- Repository APIs and examples are validated against the deployed product’s Swagger/API specification.
- Artifact bytes, metadata and approval evidence are linked by an immutable hash and auditable state transition.
- Recall can prevent new consumption, preserve evidence, identify deployed instances and track remediation.
- Backup restore, scanner outage, external-intelligence outage and false-positive scenarios are documented and tested.

## Authoritative references consulted

- GitLab Docs, **Merge request approvals**: https://docs.gitlab.com/user/project/merge_requests/approvals/
- VirusTotal Docs, **Public vs Premium API**: https://docs.virustotal.com/reference/public-vs-premium-api
- NIST NSRL, **Current RDS Hash Sets**: https://www.nist.gov/itl/csd/secure-systems-and-applications/national-software-reference-library-nsrl/nsrl-download-0
- Sonatype, **Repository Firewall** and **Firewall Quarantine**: https://help.sonatype.com/en/repository-firewall.html and https://help.sonatype.com/en/firewall-quarantine.html
- Sonatype, **Tagging**: https://help.sonatype.com/en/tagging.html
- Dependency-Track releases: https://github.com/DependencyTrack/dependency-track/releases
- CycloneDX, **Machine Learning Bill of Materials**: https://cyclonedx.org/capabilities/mlbom/
- PyTorch, **torch.load security warning**: https://docs.pytorch.org/docs/stable/generated/torch.load.html
- Microsoft Learn, **WSUS and the Microsoft Update Catalog**: https://learn.microsoft.com/en-us/windows-server/administration/windows-server-update-services/manage/wsus-and-the-catalog-site
- Aqua Security / GitHub advisory, **Trivy ecosystem supply chain temporarily compromised**: https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23
