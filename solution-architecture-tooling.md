# Solution Architecture: Tooling and Implementation Guide
## Enterprise Package Intake and Approved Repository

## Document control

| Field | Value |
|---|---|
| Document title | Solution Architecture: Tooling and Implementation Guide |
| Version | 1.5 |
| Status | Draft for review |
| Owner | Security Architecture |
| Last updated | 2026-08-02 |

### Revision history

| Version | Date | Author | Summary of changes |
|---|---|---|---|
| 1.0 | 2026-04-15 | Security Architecture | Initial draft — tooling mapped to all eleven stages; Sonatype feature map; phased implementation plan |
| 1.1 | 2026-04-18 | Security Architecture | Added paid and cloud alternatives table; integration wiring diagram |
| 1.2 | 2026-04-22 | Security Architecture | Added proprietary binary tooling path; added CMDB options; added inventory data flow summary; split Stage 5 into 5a/5b |
| 1.3 | 2026-04-23 | Security Architecture | Added Stage 11b retroactive binary recheck tooling; added binary authentication tools (Authenticode, NSRL, MalwareBazaar, VirusTotal hash); updated Stage 4 to split 4a/4b; updated tooling summary table; updated integration wiring diagram; added publisher certificate thumbprint register to CMDB requirements; updated phased rollout |
| 1.4 | 2026-08-01 | Security Architecture | Added Stage 4c/5c tooling for AI/ML model artifacts (Path C); added macOS and Linux platform signing tools alongside Authenticode; added pipeline tooling supply-chain pinning practices; added push-remediation tooling for Stage 10; added Stage 11b rate-limit budgeting, false-positive suppression schema, and pipeline-admin segregation of duties; added backup/DR tooling; added Phase 7 for Path C rollout; added "Open issues for internal review" section mirrored from the architecture document |
| 1.5 | 2026-08-02 | Security Architecture | Scoped macOS code-signing tooling as optional/build-on-demand based on confirmed enterprise estate (Windows desktops, mixed Windows/Linux servers, Linux developer IDEs); moved the macOS rollout item out of the Phase 7 critical path; Windows (Authenticode) and Linux (GPG/RPM/DEB) tooling remain baseline |

> **Scope:** This document identifies the specific tools, packages, and integration points required to implement the controlled package intake architecture. It is organised by workflow stage and includes a recommended stack, alternatives, licensing notes, and integration guidance.
>
> **Preference:** Free, self-hosted tools are the primary recommendation. Paid and commercial options are noted separately where they offer meaningful capability advantages.
>
> **Platform scope:** The confirmed enterprise estate is Windows desktops, a mix of Windows and Linux servers, and Linux developer IDEs — no macOS fleet has been confirmed. Windows (Authenticode) and Linux (GPG/RPM/DEB) tooling in Stage 4b are baseline requirements. macOS (`codesign`/notarization) tooling is documented for completeness wherever it appears in this guide but is **optional, build-on-demand** — implement it only if a macOS artifact or endpoint population actually enters scope.
>
> **Key distinction — four concerns, four toolsets:**
> - Stages 4a/5a/11a: Open-source SBOM path — Syft, Grype, Dependency-Track.
> - Stages 4b/5b: Proprietary binary vendor-advisory path — sha256sum, platform signature verification (Authenticode/GPG as baseline, codesign as optional), NSRL, vendor advisory scripts, CMDB.
> - Stages 4c/5c: AI/ML model artifact path — hub revision pinning, `safetensors`/GGUF format enforcement, pickle static-scan exception handling, model card capture, CMDB/model registry.
> - Stage 6 and 11b: Binary recheck for all artifact types — ClamAV, YARA, MalwareBazaar, VirusTotal hash API, NSRL, CAPE sandbox or model load-test sandbox. This stage addresses supply chain attacks where a trojanised binary or repointed model has a correct version string and no published CVE — a class of threat invisible to SBOM-based tools.
>
> A fifth, cross-cutting concern — the integrity of the scanning tooling itself — is addressed under "Pipeline tooling supply chain" rather than folded into any single stage.

---

## Tooling summary by stage

| Stage | Primary tool (self-hosted free) | Sonatype / Nexus Pro capability | Paid / cloud alternative |
|---|---|---|---|
| 1 · Request and approval | GitLab CE or OpenProject | Nexus IQ policy intake via API | Jira Cloud, ServiceNow |
| 2 · Restricted egress proxy | Squid 7.x + iptables/nftables | Nexus proxy repositories | Zscaler Internet Access, Palo Alto NGFW |
| 3 · Quarantine repository | Nexus Pro (quarantine capability) | **Native: Firewall Quarantine** | Artifactory Pro |
| 4a · Integrity — open source | cosign, sigstore, sha256sum, Authenticode check | Nexus IQ provenance checks | Chainguard, Sigstore Public Good |
| 4b · Integrity — proprietary | sha256sum, osslsigncode / signtool (baseline), gpg / rpm --checksig (baseline), codesign (optional, on demand), NSRL lookup | Nexus component hash attributes | — |
| 4c · Integrity — AI/ML model | huggingface_hub API (revision pinning) + fickling / picklescan + safetensors validator | Nexus component hash attributes | — |
| 5a · SBOM and SCA — open source | Syft + Grype + Dependency-Track | Sonatype Lifecycle (IQ Server) | FOSSA, Anchore Enterprise |
| 5b · Vendor advisory — proprietary | Custom pipeline script + Nexus tags + CMDB | Nexus Pro custom metadata API | ServiceNow CMDB integration |
| 5c · Model card and license capture — AI/ML | Custom pipeline script + Nexus tags + CMDB/model registry | Nexus Pro custom metadata API | MLflow Model Registry |
| 6 · Malware and sandbox | ClamAV + YARA + MalwareBazaar + NSRL + CAPE Sandbox + model load-test harness | Nexus IQ malware intelligence | Recorded Future Sandbox, ANY.RUN |
| 7 · Cooling-off delay | Custom Jenkins/GitLab pipeline gate | IQ Server policy age rules | — |
| 8 · Test pipeline | Jenkins or GitLab CI, isolated runners | — | GitHub Actions (private runners) |
| 9 · Promotion review | GitLab MR approvals or Jenkins gates | IQ Server promotion workflows | — |
| 10 · Consumption and inventory | Nexus Pro + Dependency-Track (open source) + CMDB (proprietary and model) + push-remediation via WSUS/Intune/Ansible | **Native: approved repositories** | Artifactory Pro |
| 11a · CVE recheck — open source | Dependency-Track + Prometheus/Grafana/Alertmanager | Sonatype Lifecycle continuous scan | Snyk, JFrog Xray |
| 11a · CVE recheck — proprietary | Vendor bulletin RSS + NVD CPE watches + WSUS/Intune | — | Microsoft Sentinel, Splunk |
| 11a · CVE recheck — AI/ML model | Hub API revision-drift poll + model-card watch | — | — |
| 11b · Binary recheck — all types | Scheduled job: VT hash API + MalwareBazaar + YARA + NSRL + platform signature OCSP/revocation + CAPE on IOC + model hub-revision drift; rate-limit budgeted, false-positive suppression aware | — | ReversingLabs TitaniumCloud, Recorded Future |
| Pipeline tooling trust root | Pinned tool versions + cosign signature verification on updates + canary-sample self-audit job | — | — |

---

## Stage 1 · Request and approval portal

The portal is the primary interface for teams to submit intake requests and for security reviewers to approve, reject, escalate, or set expiry on approvals. The artifact type field drives which pipeline path is executed downstream and which Stage 11b recheck controls apply.

### Recommended self-hosted free option

**GitLab Community Edition (CE)** is the preferred choice. GitLab CE is free under the MIT licence, fully self-hosted, and provides issue templates, approval rules, label-based workflow state, webhooks, and a REST API.

Key capabilities used:
- Issue templates with required fields: package name, version, source URL, checksum if available, **artifact type** (open-source / proprietary binary / firmware), justification, owner, target environment, review date.
- The artifact type field determines Path A (SBOM) or Path B (vendor advisory) and the applicable 11b recheck schedule.
- Label-based state machine for workflow stages.
- Milestone or due-date field for expiry tracking.
- RBAC for separation of duties — requestor cannot self-approve for production.
- LDAP or SSO integration via Keycloak.

**Alternative self-hosted free:** OpenProject (GNU GPL v3).

**Paid alternatives:** Jira Cloud (note: Jira Server EOL February 2024; Data Center EOL March 2029), ServiceNow.

---

## Stage 2 · Restricted egress proxy

**Squid 7.x** (open source, GPL) deployed on a hardened Linux host. Do not use the deprecated pfSense Squid package.

For proprietary binary downloads, allowlist specific vendor domains individually (e.g. `catalog.update.microsoft.com`, `download.adobe.com`) rather than broad TLD rules.

Dependency-confusion prevention for open-source: allowlist required upstream registry domains; block requests where URL path matches an internal namespace directed at a public registry; log all denied requests to OpenSearch.

**Complementary:** iptables or nftables on all endpoints blocking direct outbound TCP 443 and TCP 80 except to the Squid proxy IP.

**Alternative:** OPNsense (BSD, free) with built-in Squid proxy.

**Paid alternative:** Zscaler Internet Access or Palo Alto Networks NGFW.

---

## Stage 3 · Quarantine repository

**Nexus Repository Pro** with the Firewall Audit and Quarantine capability enabled per proxy repository.

The SHA-256 hash stored as a component attribute in Nexus is the primary key used by the Stage 11b retroactive recheck job to query VirusTotal, MalwareBazaar, and NSRL. It is critical that hashes are stored as searchable component attributes, not only as file checksums, so the recheck job can retrieve them in bulk via the Nexus REST API without downloading every binary.

Lifecycle repository groups:
- `intake-quarantine` — initial staging, no consumer access.
- `intake-dev-approved` — cleared for development use.
- `intake-prod-approved` — cleared for production use.

---

## Stage 4a · Integrity and authenticity — open-source path

**cosign** (Sigstore, Apache 2.0): sign and verify container images and arbitrary blobs; attach SLSA provenance attestations.

**sha256sum / sha512sum** (GNU Coreutils): checksum verification against publisher-provided hashes.

**in-toto** (Apache 2.0): supply chain attestation recording who fetched, scanned, and approved each artifact.

**Sigstore Rekor** (Apache 2.0): transparency log for signed attestations.

**Authenticode verification for Windows PE binaries (open-source tools):** Even open-source Windows tools such as Notepad++ publish Authenticode-signed releases. Verify the signature and compare the certificate thumbprint against the expected publisher thumbprint stored in the CMDB. Use `osslsigncode verify` (Linux) or `signtool verify` (Windows) in the pipeline.

```bash
# Linux pipeline — verify Authenticode signature using osslsigncode
osslsigncode verify -in notepadplusplus_installer.exe
# Returns exit code 0 if valid, non-zero if invalid or unsigned
```

**Dependency-confusion check scripts:** Python or Bash scripts blocking promotion on namespace collision.

**GPG / GnuPG**: verifying PGP-signed apt/rpm packages and Maven artifacts.

---

## Stage 4b · Integrity and authenticity — proprietary binary path

**sha256sum:** Compare the downloaded SHA-256 against the vendor's published catalog hash. Hard block on mismatch.

**Authenticode / code signing verification (Windows):** All Windows PE binaries must have their Authenticode signatures verified and certificate thumbprints compared against the expected publisher value stored in the CMDB publisher certificate register.

```powershell
# PowerShell — verify Authenticode on test Windows host in isolated pipeline
$sig = Get-AuthenticodeSignature -FilePath $artifactPath
if ($sig.Status -ne "Valid") { throw "Authenticode invalid: $($sig.Status)" }
$actual = $sig.SignerCertificate.Thumbprint
$expected = Get-CMDBThumbprint -Publisher $publisherName
if ($actual -ne $expected) { throw "Cert thumbprint mismatch. Expected $expected, got $actual" }
```

For Linux pipeline use `osslsigncode verify -in binary.exe` to check signature validity without a Windows host.

**Code signing and notarization verification (macOS) — optional, build on demand:** not part of the baseline rollout given the confirmed enterprise estate (Windows desktops, Windows/Linux servers, Linux developer IDEs). Documented here so the pipeline has a ready-made path if a macOS artifact or endpoint population is ever confirmed in scope; until then, skip Phase 7 item 33 below and let a submitted macOS artifact fall back to hash verification plus Tier 1 cooling-off, the same fallback already used for any unsigned vendor artifact. If built, `.pkg`, `.dmg`, and `.app` artifacts are verified with the native `codesign` and `spctl` tools, run on a macOS build agent in the pipeline (a Linux-only pipeline cannot fully verify Apple notarization tickets — a macOS build agent is itself new infrastructure this estate does not currently need).

```bash
# Verify signature validity and extract Team ID
codesign --verify --deep --strict --verbose=2 "$artifactPath"
actual_team_id=$(codesign -dv --verbose=4 "$artifactPath" 2>&1 | grep "TeamIdentifier" | cut -d= -f2)
expected_team_id=$(get_cmdb_team_id "$publisherName")
if [ "$actual_team_id" != "$expected_team_id" ]; then
    echo "Team ID mismatch. Expected $expected_team_id, got $actual_team_id"; exit 1
fi

# Verify notarization
spctl --assess --type execute --verbose "$artifactPath"
```

**Package and binary signature verification (Linux):** RPM/DEB packages use their native signature verification against an imported publisher public key; standalone ELF binaries or tarballs are verified with a detached GPG signature where the vendor provides one.

```bash
# Import the publisher's public key once, then verify per artifact
rpm --checksig vendor-package.rpm
dpkg-sig --verify vendor-package.deb
gpg --verify vendor-binary.tar.gz.sig vendor-binary.tar.gz
```

**NSRL positive-assertion lookup (Stage 4b):** Load the NIST NSRL dataset into a local PostgreSQL database and query it as part of the pipeline. A match provides positive confirmation that the hash corresponds to a known legitimate release. Absence from NSRL is not a block — niche or internal tools may not be indexed — but the result is recorded in Nexus metadata.

```python
def check_nsrl(sha256_hash: str, nsrl_db) -> dict:
    cursor = nsrl_db.execute(
        "SELECT FileName, ProductName, ProductVersion FROM NSRLFile WHERE SHA256 = ?",
        (sha256_hash.upper(),)
    )
    row = cursor.fetchone()
    if row:
        return {"known_good": True, "product": row[1], "version": row[2]}
    return {"known_good": False}
```

**Vendor catalog integration scripts:** Python scripts that query the Microsoft Update Catalog API or vendor download manifests to retrieve the expected hash for a given KB number or product version.

---

## Stage 4c · Integrity and authenticity — AI/ML model artifact path

Model artifacts have no code-signing equivalent. The authenticity control is provenance pinning against the source hub plus a mandatory serialization-format check, since the primary risk is deserialization code execution, not a tampered binary evading a signature check.

**huggingface_hub (or equivalent hub client library):** resolves and pins the exact commit SHA for a model reference at intake, rather than accepting a mutable branch or tag.

```python
from huggingface_hub import HfApi

def resolve_pinned_revision(repo_id: str, requested_ref: str) -> str:
    api = HfApi()
    info = api.model_info(repo_id, revision=requested_ref)
    return info.sha  # exact commit SHA — this is what gets pinned, not requested_ref

def get_hub_file_hashes(repo_id: str, revision: str) -> dict:
    api = HfApi()
    files = api.model_info(repo_id, revision=revision, files_metadata=True).siblings
    return {f.rfilename: f.lfs["sha256"] for f in files if f.lfs}
```

**safetensors validator:** confirms the file is a well-formed `safetensors` container with no embedded executable payload — the format's header is JSON metadata followed by raw tensor bytes, so validation is a structural parse rather than a signature check.

```python
from safetensors import safe_open

def validate_safetensors(path: str) -> bool:
    try:
        with safe_open(path, framework="numpy") as f:
            list(f.keys())  # forces header parse; raises on malformed container
        return True
    except Exception:
        return False
```

**fickling (Trail of Bits, GPL) or picklescan:** static scanners for legacy pickle-based checkpoints (`.pt`, `.pth`, `.bin`, `.ckpt`) that flag opcodes capable of arbitrary code execution (`__reduce__`, `GLOBAL` imports of `os`/`subprocess`/`eval`) rather than plain tensor-construction calls. Required whenever a checkpoint is not `safetensors` or GGUF, regardless of file extension.

```bash
# fickling — static pickle safety scan
fickling --check-safety model_checkpoint.pt

# picklescan — alternative, scans for known-dangerous globals
picklescan --path model_checkpoint.pt
```

A scan finding any code-execution-capable opcode is a hard block, not a flagged-for-review finding — this mirrors the hash-mismatch hard block in Stage 4b, not the "flag for review" pattern used for lower-confidence signals elsewhere in this pipeline.

---

## Stage 5a · SBOM generation and SCA analysis — open-source path

**Syft** (Anchore, Apache 2.0): recommended SBOM generator. Supports 20+ language ecosystems. As of March 2026, Trivy experienced two supply chain compromises and is not recommended for production CI/CD use; use Syft instead.

```bash
syft packages /path/to/artifact.tar.gz -o cyclonedx-json > artifact-sbom.json
```

**Grype** (Anchore, Apache 2.0): recommended vulnerability scanner. Accepts Syft SBOMs. Risk scoring combines CVSS severity, EPSS exploit probability, and KEV catalog status.

```bash
grype sbom:artifact-sbom.json --output json > grype-results.json
```

**OWASP Dependency-Track** (Apache 2.0): continuous SBOM management and CVE monitoring platform. Minimum 8 GB RAM. Provides portfolio-wide visibility, where-used analysis, policy enforcement, and notifications. Used only for artifacts that have a meaningful SBOM.

```bash
docker pull dependencytrack/bundled
docker volume create --name dependency-track
docker run -d -m 8192m -p 8080:8080 --name dependency-track \
  -v dependency-track:/data dependencytrack/bundled
```

**License analysis:** Grant (Anchore, Apache 2.0), FOSSology (GPL, self-hosted), or FOSSA (paid).

### Sonatype Lifecycle for open-source SBOM

When Sonatype Lifecycle (IQ Server) is licensed, it handles SBOM generation, SCA, and license analysis natively against 140M+ components, replacing Syft + Grype + Dependency-Track for the open-source path.

---

## Stage 5b · Vendor advisory capture — proprietary binary path

There is no SBOM tool for this path. A custom pipeline script captures structured vendor advisory metadata and stores it in Nexus and the CMDB.

### Pipeline script responsibilities

1. Read artifact type and vendor reference (KB number, product version) from the GitLab intake ticket.
2. Query the vendor advisory source — Microsoft MSRC API, vendor release notes — to retrieve CVEs addressed, vendor severity, and affected product scope.
3. Structure data as a JSON advisory record.
4. Write summary fields as Nexus component metadata tags: `advisory.kb_number`, `advisory.cves`, `advisory.vendor_severity`, `advisory.vendor_name`.
5. Write the Authenticode thumbprint verification result and the NSRL result as additional Nexus tags.
6. Attach the full advisory JSON to the artifact in Nexus.
7. Create or update the CMDB entry, including the expected Authenticode thumbprint for this publisher.

### Nexus Pro custom metadata API

```bash
curl -u user:pass -X PUT \
  "https://nexus.internal/service/rest/v1/components/{id}/tags" \
  -H "Content-Type: application/json" \
  -d '{
    "advisory.kb_number":"KB5034441",
    "advisory.cves":"CVE-2024-21338,CVE-2024-21345",
    "advisory.vendor_severity":"Critical",
    "auth.authenticode_status":"Valid",
    "auth.cert_thumbprint_match":"true",
    "auth.nsrl_result":"known_good",
    "auth.malwarebazaar_result":"not_found"
  }'
```

### CMDB tooling options

**ServiceNow (paid):** Full CMDB with CI records, relationship mapping, SLA, and automated discovery. Best option if already deployed.

**GitLab CE structured database (free, self-hosted):** PostgreSQL table populated via GitLab API at promotion time. The CMDB publisher certificate thumbprint register is a table in this database mapping publisher names to expected thumbprints.

**OpenProject (free, self-hosted):** Work packages with custom fields including expected thumbprint.

**Ralph (Apache 2.0, self-hosted):** Purpose-built CMDB with REST API and Docker deployment. Suitable for organisations wanting a dedicated asset management system without ServiceNow costs.

The minimum CMDB record for a proprietary artifact: artifact name and version, Nexus reference URL, GitLab ticket reference, named owner, approval expiry, **expected platform signing identity for this publisher (Authenticode thumbprint, Apple Team ID, or GPG key fingerprint)**, vendor advisory subscription reference or NVD CPE string, deployment scope, and next review date.

---

## Stage 5c · Model card and license capture — AI/ML model path

There is no SBOM tool for model artifacts. A custom pipeline script captures model card, license, and lineage metadata, mirroring the Stage 5b pattern for proprietary binaries.

### Pipeline script responsibilities

1. Read the model repo ID and requested reference from the GitLab intake ticket.
2. Fetch the model card (README/metadata) from the hub API. Parse declared license, base-model lineage (for fine-tunes and adapters), and any declared training data provenance.
3. Record the hub's verified-organisation status for the publishing namespace.
4. Structure the model card, license, and lineage data as a JSON record.
5. Write summary fields as Nexus component metadata tags: `model.hub_repo_id`, `model.pinned_revision`, `model.license`, `model.base_model`, `model.publisher_verified`.
6. Write the pickle-scan result (Stage 4c) and serialization format as additional Nexus tags.
7. Attach the full model card JSON to the artifact in Nexus.
8. Create or update the CMDB (or model registry) entry, keyed to the pinned hub revision.

### Nexus Pro custom metadata API

```bash
curl -u user:pass -X PUT \
  "https://nexus.internal/service/rest/v1/components/{id}/tags" \
  -H "Content-Type: application/json" \
  -d '{
    "model.hub_repo_id":"org/model-name",
    "model.pinned_revision":"a1b2c3d4e5f6...",
    "model.license":"apache-2.0",
    "model.base_model":"org/base-model-name",
    "model.publisher_verified":"true",
    "model.serialization_format":"safetensors",
    "model.pickle_scan_result":"not_applicable"
  }'
```

### License classification note

Model licenses are a distinct compliance surface from OSS SPDX/CycloneDX license scanning — variants such as research-only, non-commercial, and custom "responsible AI license" (RAIL) terms are common and are not always machine-readable the way an SPDX identifier is. The Stage 5c script flags any license string it cannot map to a pre-approved list for manual legal/compliance review before Stage 9 promotion, rather than defaulting to approve.

### Model registry tooling options

**Nexus tags plus CMDB (default, no new system):** consistent with the Path B pattern; sufficient for organisations primarily consuming pre-trained models rather than doing extensive in-house fine-tuning.

**MLflow Model Registry (Apache 2.0, self-hosted):** if the organisation already runs a fine-tuning or training pipeline, a dedicated model registry gives native lineage graphs (which fine-tune came from which base model, which experiment produced which checkpoint) that Nexus tags cannot represent well. This is an implementation decision — see "Open issues for internal review."

---

## Stage 6 · Malware and sandbox screening

### Static screening — all artifact types

**ClamAV** (GPL): open-source antivirus daemon applied to all artifact types.

**YARA** (BSD / VirusTotal) with community rulesets: pattern matching applied to all artifact types. Update YARA rulesets from community feeds on a regular schedule — new rules published in response to discovered campaigns are the foundation of the Stage 11b re-scan.

**MalwareBazaar** (abuse.ch, free API): hash lookup at intake. A `hash_found` result is a hard block. Not finding a hash does not confirm the file is clean.

```python
def check_malwarebazaar(sha256: str) -> bool:
    resp = requests.post("https://mb-api.abuse.ch/api/v1/",
                         data={"query": "get_info", "hash": sha256}, timeout=10)
    return resp.json().get("query_status") == "hash_found"
```

**NSRL lookup** (Stage 6 cross-check): record result in Nexus metadata. Not a blocking control on its own — absence from NSRL is not malicious — but a positive match adds confidence.

**VirusTotal hash lookup** (free tier: 500 lookups/day; no file submission): safe for all artifact types including proprietary because only the SHA-256 is transmitted. Permitted for open-source at intake; also used in Stage 11b recheck for all artifact types.

```python
def check_vt_hash(sha256: str, api_key: str) -> int:
    url = f"https://www.virustotal.com/api/v3/files/{sha256}"
    resp = requests.get(url, headers={"x-apikey": api_key}, timeout=10)
    if resp.status_code == 404:
        return -1  # Not in VT — unknown, not confirmed clean
    return resp.json()["data"]["attributes"]["last_analysis_stats"]["malicious"]
```

### Dynamic sandbox — all proprietary artifact types

**CAPE Sandbox** (GPL): recommended open-source dynamic analysis sandbox. Derived from Cuckoo v1 with automatic payload unpacking, config extraction, and YARA-based classification. Supports Windows 10 and 11 guests.

CAPE architecture: Ubuntu LTS host with KVM; Windows 10 or 11 23H2 VMs; REST API for submission and report retrieval; reports attached to Nexus artifact records.

Data handling policy defines: artifacts eligible for CAPE (all proprietary binaries, high-risk open-source); artifacts restricted to private CAPE only (proprietary ISV software); artifacts ineligible for any external sandbox (classified content).

**Paid cloud alternatives:** ANY.RUN (paid tiers for private analysis), Joe Sandbox, Recorded Future Sandbox — only for artifacts cleared for external submission.

### Isolated load-test sandbox — AI/ML model artifacts

CAPE's Windows-guest design does not fit the model threat model. Instead, the model is loaded inside a disposable, no-egress container with syscall monitoring, and the pipeline asserts that loading did not spawn a child process, write outside the working directory, or attempt an outbound connection.

```python
import subprocess, json

def isolated_model_load_test(model_path: str, loader_script: str) -> dict:
    result = subprocess.run(
        ["firejail", "--net=none", "--private", "--seccomp",
         "python3", loader_script, model_path],
        capture_output=True, timeout=120
    )
    syscall_report = parse_seccomp_audit_log()
    return {
        "load_succeeded": result.returncode == 0,
        "child_process_spawned": syscall_report.get("fork_exec_count", 0) > 0,
        "filesystem_writes_outside_workdir": syscall_report.get("oob_writes", []),
        "network_attempts": syscall_report.get("connect_attempts", []),
    }
```

`firejail` (self-hosted, GPL) or a disposable gVisor/Kata Containers sandbox both work; the requirement is no-egress networking plus syscall auditing, not a specific tool. A Docker container with `--network none` and a seccomp profile logging `execve`/`connect` syscalls is a lighter-weight equivalent if `firejail` is not already in the stack.

### Handling packed, obfuscated, or nested-archive samples

Static scanners cannot always fully unpack a sample — commercial packers/protectors (Themida, VMProtect) on proprietary installers, or model bundles distributed as nested archives. When ClamAV/YARA/Syft cannot complete a full static pass, the pipeline records the result as `inconclusive`, not `clean`, and this status forces mandatory dynamic detonation (CAPE or the model load-test harness) regardless of artifact type or hash reputation — an inconclusive static result removes static scanning as a basis for skipping or de-prioritising sandbox analysis, but does not by itself block promotion.

---

## Pipeline tooling supply chain

Syft, Grype, YARA, CAPE, `fickling`/`picklescan`, and the NSRL/MalwareBazaar client code are themselves internet-sourced software that every other stage depends on for a trustworthy verdict. This section is the tooling-level implementation of the trust-root control described in the architecture document.

**Version pinning:** every tool is installed at a pinned version, never `latest`, in the base image or provisioning script.

```dockerfile
# Pin exact versions in the pipeline base image — do not use :latest
RUN curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh \
    | sh -s -- -b /usr/local/bin v1.18.0
RUN curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh \
    | sh -s -- -b /usr/local/bin v0.85.0
```

**Signed-release verification on updates:** Syft and Grype publish cosign-signed releases; the update job verifies the signature before installing a new pinned version.

```bash
cosign verify-blob --key anchore-cosign.pub \
  --signature syft_1.19.0_checksums.txt.sig \
  syft_1.19.0_checksums.txt
```

**Ruleset staging:** YARA and CAPE community rule feeds (Elastic, VirusTotal community rules) are pulled into a staging directory and diffed before promotion into the production ruleset, so a rule change is reviewable the same way a code change is.

```bash
git -C /etc/yara-staging/elastic pull
diff -r /etc/yara-staging/elastic /etc/yara/elastic > ruleset-diff-$(date +%Y%m%d).txt
# Manual or automated review of the diff gates promotion into /etc/yara/elastic
```

**Canary self-audit job:** a small, fixed set of known-malicious (from a public malware research corpus, handled per the CAPE data-handling policy) and known-clean reference samples is re-run through the full Stage 6 and Stage 11b pipeline on a schedule. A canary sample that stops flagging as malicious pages the pipeline-admin on-call rotation — it means the scanning tooling regressed or was tampered with, not that the sample changed.

```bash
# Nightly canary run, alerts via Alertmanager on any unexpected verdict change
python3 canary_audit.py --canary-set /opt/canary-samples/manifest.json \
  --expect-results /opt/canary-samples/expected-verdicts.json
```

---

## Stage 7 · Cooling-off delay

**Jenkins Pipeline** or **GitLab CI/CD** delay gate:

```python
def check_delay_gate(artifact_id, risk_tier):
    quarantine_date = nexus_api.get_quarantine_timestamp(artifact_id)
    delay_days = {"tier1": 30, "tier2": 14, "tier3": 7}[risk_tier]
    if (today - quarantine_date).days < delay_days:
        raise GateFailure(f"Cooling-off period not met: {delay_days} days required")
```

Risk tier delays: Tier 1 (privileged binaries, drivers, base images, proprietary by default): 30 days. Tier 2 (runtime dependencies, unknown publishers, commercial ISV): 14 days. Tier 3 (well-established open-source, Microsoft Patch Tuesday — by explicit policy only): 7 days.

Emergency override requires a signed exception record with a named approver, justification, and expiry date.

---

## Stage 8 · Test pipeline

**Jenkins** (MIT) or **GitLab CI/CD runners** (self-managed): isolated Docker agents or VM snapshots with no internet access.

Test steps: pull artifact from quarantine; install in isolated environment; for proprietary installers log all file system changes, registry writes, network connection attempts, and service registrations; run functional tests; sign evidence with cosign; report to Jenkins/GitLab and Dependency-Track (open-source only); block promotion if evidence cannot be produced.

---

## Stage 9 · Promotion review

**GitLab CE Merge Request approvals**: MR requires sign-off from security reviewer and named artifact owner.

For proprietary path the MR description confirms: vendor advisory record complete in Nexus; CMDB entry created including expected Authenticode thumbprint; vendor bulletin subscription reference documented.

Custom promotion script: verifies approval expiry; copies artifact within Nexus (not a re-download); attaches all metadata; signs promotion record with cosign; triggers CMDB update.

---

## Stage 10 · Consumption and inventory

### Open-source — Nexus Pro plus Dependency-Track

```ini
# pip
[global]
index-url = https://nexus.internal/repository/pypi-approved/simple/
trusted-host = nexus.internal

# npm
registry=https://nexus.internal/repository/npm-approved/

# Maven (settings.xml)
<mirror>
  <id>nexus-approved</id>
  <mirrorOf>*</mirrorOf>
  <url>https://nexus.internal/repository/maven-approved/</url>
</mirror>

# apt
deb https://nexus.internal/repository/apt-approved focal main
```

Enforce lockfiles and digest pinning. Dependency-Track provides deployment inventory via where-used SBOM analysis.

### Proprietary — Nexus Pro plus CMDB plus WSUS/SCCM/Intune

Nexus stores the approved binary. CMDB holds deployment inventory and the publisher certificate thumbprint register. WSUS, SCCM, or Intune handles deployment and compliance for Microsoft patches.

### AI/ML model — Nexus Pro plus CMDB/model registry, keyed to pinned revision

CMDB (or a dedicated model registry) records deployment scope keyed to the pinned hub revision SHA, so a consuming application running the reviewed revision can be distinguished from one that resolved the model directly against the hub outside this pipeline.

### Push remediation tooling on recall

Recall workflows query deployment records to find systems with a recalled artifact already installed, then trigger remediation through whichever deployment tool already manages that system, rather than stopping at a notification.

```bash
# WSUS/SCCM: query compliance for the specific KB, then trigger removal/patch task
Get-WsusComputer -UpdateApprovalAction NotApproved -UpdateId $recalledKbGuid

# Intune: targeted script push to devices reporting the recalled app installed
az rest --method post --uri "https://graph.microsoft.com/beta/deviceManagement/..." \
  --body '{"deviceIds": ["..."], "scriptId": "remediate-recalled-artifact"}'

# Configuration-managed fleets (Ansible): targeted playbook against the CMDB-reported host list
ansible-playbook remediate-recalled-artifact.yml -i recalled_hosts.ini \
  -e "artifact_hash=$RECALLED_SHA256"
```

Where no push mechanism reaches a given system class (unmanaged workstations, air-gapped segments), the recall workflow still opens a tracked remediation ticket against the named owner with an SLA — this residual gap is one of the items in "Open issues for internal review."

---

## Stage 11a · Continuous CVE recheck

### Open-source artifacts

**OWASP Dependency-Track** continuously re-evaluates stored SBOMs against NVD, GitHub Advisories, OSV, and other feeds. New CVE matches trigger notifications via Alertmanager to Mattermost, email, or Slack, and create GitLab recall issues.

**Prometheus + Grafana + Alertmanager** (all Apache 2.0): operational dashboards and alert routing.

### Proprietary artifacts

**Microsoft MSRC RSS feed** and **vendor security bulletin subscriptions**: primary monitoring channel. CMDB entries must record the subscription reference for each approved product.

**NVD CPE watches**: configure product watches for each approved proprietary product. Route new CVE alerts via webhook to Alertmanager.

**WSUS / SCCM / Intune compliance dashboards**: show which systems are running unpatched versions.

### AI/ML model artifacts

There is no CVE-equivalent feed for model weights. Monitoring relies on the Stage 11b hub-revision-drift recheck (below) plus, where the hub exposes one, a poll of the model card's own revision history or community discussion feed for the repo. This is a thinner monitoring channel than CVE/NVD, and is called out explicitly as a residual limitation rather than implied to have equivalent coverage.

---

## Stage 11b · Retroactive binary recheck

This stage is new as of v1.3 of this document. It addresses supply chain attacks where a binary approved months ago is later identified as malicious — a class of threat that Dependency-Track and SBOM-based tools cannot detect because they rely on CVE matching against component version strings.

The recheck job is a scheduled Python or Bash script deployed alongside the pipeline infrastructure. It runs nightly for Tier 1 artifacts (developer toolchain, privileged binaries) and weekly for all others.

### Required tooling

**Nexus REST API:** The recheck job uses the Nexus component search API to extract all stored SHA-256 hashes in bulk without downloading the binaries:

```bash
# Retrieve all component hashes from Nexus approved repositories
curl -u user:pass \
  "https://nexus.internal/service/rest/v1/search?repository=intake-prod-approved" \
  | jq -r '.items[] | {name: .name, version: .version, sha256: .assets[0].checksum.sha256}'
```

**VirusTotal hash API** (free tier: 500 lookups/day; paid tier: higher rate limits for bulk queries):

```python
def recheck_vt(hashes: list, api_key: str) -> dict:
    results = {}
    for sha256 in hashes:
        url = f"https://www.virustotal.com/api/v3/files/{sha256}"
        resp = requests.get(url, headers={"x-apikey": api_key}, timeout=10)
        if resp.status_code == 200:
            malicious = resp.json()["data"]["attributes"]["last_analysis_stats"]["malicious"]
            results[sha256] = malicious
        else:
            results[sha256] = -1  # Not in VT database
    return results
```

**MalwareBazaar bulk query** (free API):

```python
def recheck_malwarebazaar(sha256: str) -> bool:
    resp = requests.post("https://mb-api.abuse.ch/api/v1/",
                         data={"query": "get_info", "hash": sha256}, timeout=10)
    return resp.json().get("query_status") == "hash_found"
```

**YARA re-scan with updated rulesets:** Pull binaries from Nexus and re-scan with the latest community YARA rules. Update rulesets from Elastic YARA rules, VirusTotal community rules, and other feeds on a weekly basis. A YARA rule published today for a campaign discovered this week will match binaries approved six months ago if those binaries were part of the same campaign.

```bash
# Update YARA rulesets
git pull https://github.com/elastic/protections-artifacts /etc/yara/elastic/
# Pull approved binaries and re-scan
nexus-cli download-all --repo intake-prod-approved --dest /tmp/recheck/
yara -r /etc/yara/ /tmp/recheck/ > /var/log/yara-recheck-$(date +%Y%m%d).log
rm -rf /tmp/recheck/  # Clean up after scan
```

**NSRL recheck** (local PostgreSQL database, free): update the NSRL dataset periodically from NIST. Recheck all hashes against the updated dataset.

**Authenticode OCSP recheck for Windows PE binaries:** Verify that the Authenticode signing certificate used on each approved Windows binary has not been subsequently revoked. Certificate revocation is a strong signal of a compromised build infrastructure or signing key.

```bash
# Check certificate revocation status via OCSP using OpenSSL
openssl ocsp \
  -issuer issuer.pem \
  -cert signing_cert.pem \
  -url http://ocsp.digicert.com \
  -text
# Extract signing cert from PE binary first using osslsigncode:
osslsigncode extract -in binary.exe -certs signing_certs.pem
```

**Targeted CAPE recheck on IOC alerts:** When a threat intelligence report publishes IoCs (file hashes, process names, network indicators), the recheck job cross-references the IoCs against the Nexus inventory. Any artifact matching an IoC is submitted to the private CAPE sandbox for re-detonation before the recall decision is made.

**Model hub revision-drift recheck:** For Path C artifacts, re-resolve the current commit SHA for the reference the artifact was originally pinned from and compare it to the pinned SHA stored at intake.

```python
from huggingface_hub import HfApi

def check_revision_drift(repo_id: str, original_ref: str, pinned_sha: str) -> dict:
    api = HfApi()
    current_sha = api.model_info(repo_id, revision=original_ref).sha
    return {
        "drifted": current_sha != pinned_sha,
        "pinned_sha": pinned_sha,
        "current_hub_sha": current_sha,
    }
```

A drift result does not itself indicate the artifact stored in Nexus has changed — Nexus holds the immutable pinned file — but it flags that anything resolving the model directly against the hub (bypassing this pipeline) would now get a different, unreviewed file.

### Rate-limit budgeting

VirusTotal's free tier caps at 500 lookups/day; MalwareBazaar and NSRL have their own practical bulk-query limits. The recheck job allocates its daily query budget by risk tier so coverage degrades predictably as inventory grows, instead of silently dropping artifacts from coverage.

```python
def build_recheck_batch(inventory: list, daily_vt_budget: int = 500) -> dict:
    tier1 = [a for a in inventory if a["risk_tier"] == "tier1"]
    remaining_budget = daily_vt_budget - len(tier1)
    tier2_3 = sorted(
        [a for a in inventory if a["risk_tier"] != "tier1"],
        key=lambda a: a["last_recheck_date"]  # oldest-checked-first rotation
    )
    return {
        "vt_this_run": tier1 + tier2_3[:max(remaining_budget, 0)],
        "deferred_to_next_run": tier2_3[max(remaining_budget, 0):],
    }
```

Whether this budgeting is sufficient long-term, or the paid VT tier becomes necessary at a given inventory size, is an implementation decision — see "Open issues for internal review."

### False-positive suppression

A finding an analyst dispositions as a confirmed false positive is written to the recheck datastore with that disposition and a suppression key (hash + signal type). Subsequent runs check the suppression table before opening a new GitLab issue for the same hash/signal combination.

```python
def should_suppress(sha256: str, signal_type: str, rule_version: str, suppression_db) -> bool:
    row = suppression_db.execute(
        """SELECT rule_version_at_disposition FROM fp_suppressions
           WHERE sha256 = ? AND signal_type = ?""",
        (sha256, signal_type)
    ).fetchone()
    if row is None:
        return False
    # Suppression only holds while the matching rule/signature is unchanged
    return row[0] == rule_version
```

### Segregation of duties for the recheck job itself

Scheduling configuration, rate-limit budget allocation, false-positive disposition authority, and YARA ruleset promotion (from the staging diff described under "Pipeline tooling supply chain") are restricted to a role distinct from artifact approvers. The recommended enforcement mechanism is routing all changes to this configuration through a GitLab merge request in the recheck-job's own repository, requiring a second approver — this gives a uniform, auditable trail regardless of how granular each underlying tool's native RBAC is. Whether native per-tool RBAC should additionally be configured is an implementation decision — see "Open issues for internal review."

### Recheck job result handling

| Signal | Automated action | Human action required |
|---|---|---|
| MalwareBazaar hash_found | Immediate block in Nexus (403); GitLab recall issue opened | Confirm block; notify affected system owners; remediate |
| VirusTotal 2+ engine detections | GitLab recall issue opened; artifact flagged but not yet blocked | Security analyst reviews; decides block or false-positive |
| VirusTotal 1 engine detection | Nexus tag updated; logged | Low priority human review |
| YARA rule match on updated ruleset | GitLab recall issue opened; flagged for review | Security analyst reviews rule context; decides action |
| Authenticode certificate revoked (OCSP) | Immediate block in Nexus; GitLab recall issue opened | Confirm block; notify affected systems; obtain clean replacement |
| Authenticode / codesign / GPG signing identity changed | GitLab recall issue opened | Analyst verifies whether publisher legitimately rotated their cert/key |
| NSRL status changed | Log entry created | Low priority human review |
| CAPE IOC match on targeted recheck | Immediate block; GitLab recall issue opened | Confirm block; incident response process initiated |
| Model hub pinned revision drifted from current hub HEAD | GitLab recall issue opened | Analyst confirms whether the Nexus-pinned file remains the intended reviewed version |
| Finding matches a suppressed confirmed-false-positive | Recheck timestamp updated; no new issue opened | None — re-evaluates automatically if hash or rule changes |
| No signals | Recheck timestamp updated in Nexus tag | No action required |

### Recheck job datastore

All recheck verdicts are written to a dedicated PostgreSQL table:

```sql
CREATE TABLE artifact_recheck_log (
    id              SERIAL PRIMARY KEY,
    sha256          TEXT NOT NULL,
    nexus_artifact  TEXT NOT NULL,
    recheck_date    TIMESTAMP NOT NULL,
    vt_malicious    INTEGER,       -- -1 = not in VT, 0 = clean, N = engine count
    mb_found        BOOLEAN,
    yara_match      TEXT,          -- rule name if matched, NULL if clean
    nsrl_status     TEXT,          -- known_good / not_indexed / status_changed
    auth_signature  TEXT,          -- Good / Revoked / Unknown / Mismatch (Authenticode, codesign, or GPG)
    cape_ioc_match  BOOLEAN,
    model_hub_drift BOOLEAN,       -- NULL for non-model artifacts
    action_taken    TEXT,          -- blocked / flagged_review / clean / false_positive
    analyst_notes   TEXT
);

CREATE TABLE fp_suppressions (
    sha256                      TEXT NOT NULL,
    signal_type                 TEXT NOT NULL,   -- e.g. 'yara', 'vt', 'nsrl'
    rule_version_at_disposition TEXT NOT NULL,
    dispositioned_by            TEXT NOT NULL,
    dispositioned_date          TIMESTAMP NOT NULL,
    justification               TEXT NOT NULL,
    PRIMARY KEY (sha256, signal_type)
);
```

These tables are queryable for audit purposes: "show me all artifacts that have been rechecked in the last 30 days and their verdicts," "show me everything that was blocked by the recheck job in the last year," or "show me every suppressed false positive and who dispositioned it."

---

## Supporting infrastructure tools

| Tool | Purpose | Licence | Deployment |
|---|---|---|---|
| **HashiCorp Vault** (BSL 1.1) or **OpenBao** (MPL 2.0) | Secrets management: Nexus credentials, API tokens, signing keys, VirusTotal API key | Free self-hosted | Docker / Kubernetes |
| **Keycloak** | SSO / OIDC identity provider | Apache 2.0 | Docker / Kubernetes |
| **OpenLDAP** | LDAP directory | OpenLDAP Public Licence | Linux package |
| **OpenSearch** | Log aggregation: Squid access logs, Nexus audit logs, pipeline events, sandbox reports, recheck job logs | Apache 2.0 | Docker Compose |
| **Mattermost** | Self-hosted team messaging for alerts and recall notifications | MIT (free tier) | Docker |
| **Cosign / Sigstore** | Artifact and SBOM signing, attestation | Apache 2.0 | CLI, Docker |
| **MinIO** | S3-compatible object storage for SBOM archive, test evidence, sandbox reports, YARA ruleset archive | AGPL 3.0 | Docker / Kubernetes |
| **PostgreSQL** | Backing database for Dependency-Track, GitLab, Jenkins, Keycloak, NSRL dataset, recheck log | PostgreSQL Licence | Docker / system package |
| **Ralph** | Open-source CMDB for proprietary software inventory including publisher cert thumbprint register | Apache 2.0 | Docker |
| **osslsigncode** | Cross-platform Authenticode signature verification (Linux pipeline use) | GPL | Linux package (`apt install osslsigncode`) |
| **fickling** | Static safety scanner for pickle-based model checkpoints (Path C exception path) | GPL / LGPL (Trail of Bits) | `pip install fickling` |
| **safetensors** | Reference library for validating and reading the required model serialization format | Apache 2.0 | `pip install safetensors` |
| **firejail** or **gVisor** | No-egress, syscall-audited sandbox for the AI/ML model load-test harness | GPL / Apache 2.0 | Linux package / container runtime |
| **pgBackRest** or **restic** | Backup tooling for PostgreSQL-backed systems of record (CMDB, recheck job datastore, NSRL DB) and Nexus blob storage respectively | MIT / BSD-2 | Cron-scheduled job |
| **MLflow Model Registry** (optional) | Dedicated model lineage tracking if in-house fine-tuning volume justifies a system beyond Nexus tags plus CMDB | Apache 2.0 | Docker / Kubernetes |

---

## Inventory data flow summary

### At intake time (written by pipeline)

```
GitLab intake ticket created by requestor
    → Approval granted by security reviewer (expiry date set)
    → Webhook fires to GitLab CI / Jenkins
    → Squid proxy fetches artifact from allowlisted source
    → Artifact lands in Nexus quarantine repo with SHA-256 stored as component attribute
    → Pipeline writes initial Nexus tags:
          requestor, approver, expiry, GitLab ticket ref, risk tier, artifact type

    For open source (Path A):
        Syft generates SBOM → uploaded to Dependency-Track + attached to Nexus artifact
        Grype scans SBOM → findings uploaded to Dependency-Track
        Authenticode checked if Windows PE → result tagged in Nexus
        ClamAV + YARA + MalwareBazaar + NSRL + VT hash lookup → results tagged in Nexus
        CAPE for high-risk samples → report attached to Nexus artifact

    For proprietary (Path B):
        sha256sum verified against vendor catalog → result tagged
        Platform signature validity checked (Authenticode/codesign/GPG) → result tagged
        Signing identity extracted and compared against CMDB expected value → result tagged
        NSRL lookup → result tagged
        MalwareBazaar hash lookup → result tagged
        VirusTotal hash lookup → result tagged
        Advisory record fetched → KB/CVE/severity tagged in Nexus + advisory JSON attached
        CMDB record created including expected signing identity, advisory reference
        ClamAV + YARA → results tagged
        CAPE private sandbox (mandatory) → report attached to Nexus artifact

    For AI/ML model (Path C):
        Hub commit SHA resolved and pinned (not a mutable branch/tag) → result tagged
        Downloaded SHA-256 compared against hub-published file hash → result tagged
        Serialization format validated (safetensors/GGUF) or routed to pickle exception scan → result tagged
        fickling/picklescan static scan run on any non-safetensors/GGUF file → hard block on unsafe opcodes
        Model card, license, and base-model lineage fetched → JSON attached + summary tagged
        Publisher/org verification status recorded
        MalwareBazaar hash lookup → result tagged
        ClamAV + YARA → results tagged
        Isolated no-egress load-test sandbox (mandatory) → report attached to Nexus artifact
        CMDB/model registry record created including pinned revision SHA

    → Delay gate enforced (checks quarantine timestamp against risk tier policy)
    → Test pipeline runs in isolated environment; signed evidence attached to Nexus
    → Promotion MR raised in GitLab; both reviewers must approve
    → On approval: artifact promoted in Nexus repo groups; CMDB entry updated
```

### During Stage 11b recheck (scheduled job)

```
Nightly / weekly recheck job:
    → Nexus REST API: extract all SHA-256 hashes from approved repo groups
    → Build recheck batch: Tier 1 every run, Tier 2/3 rotated within remaining rate-limit budget
    → For each hash in this run's batch:
        Check fp_suppressions table: skip alerting (but still record check) if suppressed and rule unchanged
        VirusTotal hash API: check current detection count (within daily budget)
        MalwareBazaar API: check if now indexed as malicious
        Pull binary from Nexus to temp storage
        YARA re-scan with updated rulesets
        NSRL recheck with updated dataset
        For signed binaries: extract signing identity, check revocation status (OCSP/CRL by platform)
        For model artifacts: re-resolve hub HEAD for original reference, compare to pinned SHA
        Delete temp binary after scan
    → Compare against known IoCs from latest threat intel reports
    → For any hits: trigger CAPE targeted recheck (or model load-test re-run)
    → Write all verdicts to recheck_log PostgreSQL table
    → Update Nexus component tags with latest recheck timestamp and result
    → For any unsuppressed blocking signals: call Nexus API to block artifact; open GitLab recall issue
    → Query CMDB/WSUS/Intune/Dependency-Track for known-deployed instances of blocked artifact
    → Trigger push remediation via available deployment tool; open manual-follow-up ticket where none exists
    → Alertmanager routes recall notifications to named owners
```

### During incident response (queried by security team)

```
CVE found in open-source component (Stage 11a):
    Dependency-Track: which projects use affected component?
    Nexus: download specific approved version; view intake metadata

Post-approval compromise detected by recheck job (Stage 11b):
    recheck_log table: when was this artifact last checked; what triggered the alert?
    fp_suppressions table: was a prior related finding on this hash dispositioned, and by whom?
    Nexus component tags: what were the intake-time verdicts for comparison?
    GitLab recall issue: who is the named owner; which systems are affected?
    For open source: Dependency-Track where-used analysis
    For proprietary: CMDB deployment list + WSUS/Intune compliance status
    For model artifacts: CMDB deployment list keyed to pinned revision SHA

Specific threat intel received about a tool in use:
    Query Nexus by tool name and version for stored SHA-256
    Cross-reference against published IoC hash list
    If match: trigger immediate CAPE recheck; open GitLab recall issue
    Query CMDB or Dependency-Track for deployment scope

Suspected compromise of the pipeline tooling itself (Pipeline tooling supply chain):
    canary_audit job history: did a canary sample's verdict change unexpectedly?
    Tool version pin history (base image / provisioning script git log): what changed and when?
    cosign verification log: did any tooling update fail signature verification and get overridden?
```

---

## Complete recommended self-hosted free stack

```
Stage 1    GitLab CE                    — request portal, approval workflow, MR promotion gates
Stage 2    Squid 7.x + nftables         — egress proxy, dep-confusion enforcement, access logging
Stage 3    Nexus Repository Pro         — quarantine repo (Pro licence); SHA-256 stored as searchable attribute
Stage 4a   cosign + sha256sum + in-toto + osslsigncode  — open-source integrity, Authenticode, attestation
Stage 4b   sha256sum + osslsigncode + gpg/rpm --checksig (baseline) + codesign (optional, on demand) + NSRL + vendor catalog scripts  — proprietary hash, platform signature, NSRL
Stage 4c   huggingface_hub + safetensors + fickling/picklescan  — model revision pinning, format safety
Stage 5a   Syft + Grype + Dependency-Track  — SBOM, SCA, continuous CVE monitoring
Stage 5b   Custom pipeline script + Nexus tags + CMDB  — vendor advisory capture and inventory
Stage 5c   Custom pipeline script + Nexus tags + CMDB/MLflow  — model card, license, lineage capture
Stage 6    ClamAV + YARA + MalwareBazaar + NSRL + VT hash + CAPE Sandbox + firejail/gVisor load-test  — all artifact types
Stage 7    Jenkins / GitLab CI gate     — cooling-off delay enforcement
Stage 8    Jenkins / GitLab CI + Docker isolated runners  — test pipeline
Stage 9    GitLab CE MR approvals + cosign  — promotion review gate with sign-off
Stage 10   Nexus Pro + Dependency-Track (open source) + CMDB + WSUS/Intune/Ansible (proprietary and model, incl. push remediation)
Stage 11a  Dependency-Track + vendor bulletin RSS + NVD CPE watches + hub revision poll + Alertmanager
Stage 11b  Scheduled recheck job: VT hash API + MalwareBazaar + YARA + NSRL + signature OCSP + CAPE on IoC + hub-revision drift
           Rate-limit budgeting and fp_suppressions table for coverage-at-scale and alert-fatigue control
           Recheck log: PostgreSQL tables for audit history of all verdicts and suppression dispositions
Trust root Pinned tool versions + cosign-verified updates + staged ruleset diffs + canary self-audit job

Supporting:
           Keycloak (SSO, incl. non-human service accounts), OpenBao (secrets), OpenSearch (logs),
           Mattermost (notifications), MinIO (artifact storage), PostgreSQL (databases + NSRL + recheck log),
           Ralph or ServiceNow (CMDB with publisher trust register), osslsigncode (Authenticode on Linux),
           pgBackRest/restic (backup for CMDB, recheck datastore, Nexus blob store)
```

---

## Sonatype platform feature map (Nexus Pro + Lifecycle)

| Architecture control | Sonatype native capability | Replaces / complements |
|---|---|---|
| Quarantine (Stage 3) | Repository Firewall Quarantine (IQ Server + Nexus Pro) | Replaces custom quarantine scripts |
| Dep-confusion prevention (Stage 2/4a) | IQ Server namespace confusion analysis | Complements Squid ACL rules |
| SBOM generation (Stage 5a) | Sonatype SBOM Manager (IQ Server) | Complements or replaces Syft |
| SCA / CVE analysis (Stage 5a) | Sonatype Lifecycle — 140M+ component database, EPSS, KEV | Complements or replaces Grype + Dependency-Track |
| License analysis (Stage 5a) | Sonatype Lifecycle — 2,000+ license threat categorisations | Replaces Grant / FOSSology |
| Vendor advisory capture (Stage 5b) | Not natively supported — custom script always required | Custom script required regardless |
| Model card and lineage capture (Stage 5c) | Not natively supported — custom script always required | Custom script or dedicated model registry (MLflow) required regardless |
| Malware intelligence (Stage 6) | IQ Server malware intelligence (AI + human threat intel) | Complements ClamAV/YARA; does not replace sandbox, hash lookups, or the model load-test harness |
| Continuous CVE recheck (Stage 11a) | Sonatype Lifecycle continuous evaluation and notifications | Complements Dependency-Track; does not cover model artifacts |
| Retroactive binary recheck (Stage 11b) | Not provided — custom scheduled job always required | Custom job always required regardless of commercial tooling |
| Pipeline tooling supply-chain integrity | Not provided — the platform vendor's own tooling is a separate trust question | Applies equally to Sonatype's own scanning components; canary self-audit still recommended |
| Policy engine | IQ Server — 18 default policies + 30+ customisable rules | Unifies Stages 3–9 for open-source path; does not cover Path C |

Note: Stage 11b retroactive binary recheck and the AI/ML model artifact path (Stages 4c/5c) are not provided by any current commercial repository or SCA platform surveyed for this document. Both require a custom scheduled job or pipeline script regardless of whether Sonatype Lifecycle, JFrog Xray, or another commercial tool is deployed.

---

## Paid and cloud tool options

| Control area | Paid tool | Differentiator vs. free option |
|---|---|---|
| Repository + SCA | **JFrog Artifactory Pro + Xray** | Multi-site replication, SBOM policy engine, native SCA |
| SCA / SBOM | **FOSSA** | Attorney-reviewed license database, SBOM export for procurement |
| SCA / SBOM | **Anchore Enterprise** | Commercial Syft + Grype with policy management and SSO |
| SCA / SBOM | **Snyk** | Developer-first IDE and CI integration, strong remediation guidance |
| Binary recheck / threat intel | **ReversingLabs TitaniumCloud** | Hash reputation, file analysis, and supply chain attack intelligence at commercial scale; replaces or augments VT + MalwareBazaar |
| Binary recheck / threat intel | **Recorded Future** | Threat intelligence correlation including supply chain attack campaigns; IoC feeds for Stage 11b targeted recheck |
| Dynamic sandbox | **ANY.RUN** | Interactive browser-based sandbox; paid tiers for private analysis |
| Dynamic sandbox | **Joe Sandbox** | Deep static and dynamic analysis with YARA integration |
| Egress proxy / CASB | **Zscaler Internet Access** | Cloud-native zero-trust proxy, dep-confusion rules, DLP |
| Egress proxy | **Palo Alto Networks NGFW** | On-premises NGFW with URL filtering and App-ID |
| CMDB | **ServiceNow** | Full CMDB with ITSM integration, SLA, automated discovery, publisher cert management |
| SIEM | **Splunk** | Centralised log analysis, correlation rules, and recall alerting |
| SIEM | **Microsoft Sentinel** | Cloud-native SIEM; integrates with Defender for DevOps and Intune |

---

## Integration wiring summary

```mermaid
flowchart TB
    GL[GitLab CE\nRequest Portal + CI/CD\nHuman and non-human requestors] -->|Webhook on approval| JK[Jenkins or GitLab CI\nPipeline Orchestrator]
    JK -->|Authorised fetch via| SQ[Squid 7.x\nEgress Proxy]
    SQ -->|Downloads to| NQ[Nexus Pro\nQuarantine Repo\nSHA-256 stored as searchable attribute]
    NQ -->|Open-source path| SCA[Path A Pipeline\nSyft + Grype + Authenticode\n+ ClamAV + YARA + MalwareBazaar + NSRL + VT hash]
    NQ -->|Proprietary path| SCB[Path B Pipeline\nsha256sum + Authenticode/codesign/GPG\n+ NSRL + Advisory capture\n+ ClamAV + YARA + MalwareBazaar + VT hash]
    NQ -->|Model path| SCC[Path C Pipeline\nHub revision pin + safetensors/GGUF check\n+ fickling/picklescan + model card capture\n+ ClamAV + YARA + MalwareBazaar]
    SCA -->|SBOM upload| DT[Dependency-Track\nOpen-source CVE monitoring]
    SCB -->|Advisory record + signing identity| CM[CMDB\nProprietary + model inventory\nPublisher trust register]
    SCC -->|Model card + pinned revision| CM
    SCA -->|High-risk samples| CS[CAPE Sandbox\nPrivate dynamic analysis]
    SCB -->|All proprietary samples| CS
    SCC -->|All model samples| MLT[No-egress Load-Test Sandbox\nfirejail / gVisor]
    SCA -->|Results + delay gate| JK
    SCB -->|Results + delay gate| JK
    SCC -->|Results + delay gate| JK
    JK -->|Isolated test| TRN[Test Runners\nDocker agents, no internet]
    TRN -->|Sign evidence| COS[cosign / in-toto\nAttestation]
    COS -->|MR approval gate| GL
    GL -->|Promote on approval| NP[Nexus Pro\nApproved Repos]
    NP -->|Open-source consumers| CO[CI/CD, Desktops, Servers]
    NP -->|Proprietary/model: feed deployment| WS[WSUS / SCCM / Intune / Ansible]
    DT -->|CVE alerts| AM[Alertmanager]
    CM -->|Vendor bulletin / hub advisory alerts| AM

    TT[Pipeline Tooling Trust Root\nPinned versions + cosign verify\n+ staged ruleset diff + canary audit] -.->|gates| SCA
    TT -.->|gates| SCB
    TT -.->|gates| SCC
    TT -.->|gates| RJ

    NP -->|11b: bulk hash extract via REST API| RJ[Recheck Job\nScheduled nightly/weekly\nRate-limit budgeted]
    RJ -->|Hash lookup| VT[VirusTotal hash API]
    RJ -->|Hash lookup| MB[MalwareBazaar API]
    RJ -->|Binary pull + scan| YR[YARA re-scan\nUpdated rulesets]
    RJ -->|Hash lookup| NS[NSRL local DB]
    RJ -->|Signature check| OC[OCSP / CRL\nAuthenticode, codesign, GPG revocation]
    RJ -->|Revision check| HD[Hub Revision-Drift Check\nModel artifacts only]
    RJ -->|IoC match: re-detonate| CS
    RJ -->|Suppression check| FP[fp_suppressions table]
    VT --> RJ
    MB --> RJ
    YR --> RJ
    NS --> RJ
    OC --> RJ
    HD --> RJ
    RJ -->|Write verdicts| RL[Recheck Log\nPostgreSQL table]
    RJ -->|Unsuppressed blocking hits| AM
    AM -->|Notifications| MM[Mattermost / Email / Slack]
    AM -->|Create recall issue| GL
    AM -->|Push remediation task| WS
    NP -->|Audit logs| OS[OpenSearch\nLog Aggregation]
    OS -->|Dashboards| GR[Grafana + Prometheus]

    RL -.->|Backup| BK[pgBackRest / restic\nDR for systems of record]
    CM -.->|Backup| BK
    NP -.->|Backup| BK
```

---

## Minimum viable implementation order

**Phase 1 — Foundation (weeks 1–4)**
1. Deploy GitLab CE. Create intake issue templates with artifact type field.
2. Deploy Squid 7.x with deny-by-default egress. Allowlist vendor domains for proprietary downloads.
3. Enable Nexus Pro Firewall Quarantine. Configure SHA-256 stored as searchable component attribute.
4. Configure RBAC, SSO (Keycloak), and MFA.

**Phase 2 — Open-source scanning (weeks 5–8)**
5. Deploy Syft + Grype as pipeline steps.
6. Deploy OWASP Dependency-Track. Upload initial SBOMs for existing approved open-source packages.
7. Deploy ClamAV + YARA. Integrate MalwareBazaar and VirusTotal hash lookups.
8. Enable cosign for artifact signing.
9. Set up NSRL local PostgreSQL database with initial dataset download.

**Phase 3 — Proprietary binary pipeline (weeks 9–12)**
10. Deploy osslsigncode on pipeline hosts. Write Stage 4b Authenticode verification script.
11. Stand up CMDB (Ralph or GitLab PostgreSQL table). Create publisher certificate thumbprint register with initial entries for known publishers (Microsoft, Adobe, Oracle, internal tooling).
12. Write Stage 4b vendor hash verification script with Microsoft Update Catalog API integration.
13. Write Stage 5b vendor advisory capture script with Microsoft MSRC API integration.
14. Configure WSUS / SCCM / Intune to pull from Nexus approved repository.

**Phase 4 — Full pipeline (weeks 13–18)**
15. Deploy CAPE Sandbox on dedicated KVM host. Integrate with pipeline for mandatory proprietary artifact detonation.
16. Implement cooling-off delay gate.
17. Configure isolated Docker test runners.
18. Implement GitLab MR promotion review workflow.
19. Deploy Prometheus + Grafana + Alertmanager. Configure Dependency-Track and vendor bulletin RSS alert routing.

**Phase 5 — Stage 11b recheck job (weeks 19–22)**
20. Build the Stage 11b recheck job script. Connect to Nexus REST API for bulk hash extraction.
21. Integrate VirusTotal hash API, MalwareBazaar API, and local NSRL database.
22. Deploy updated YARA ruleset pull from community feeds on weekly schedule.
23. Implement Authenticode OCSP recheck using osslsigncode and OpenSSL.
24. Create the `artifact_recheck_log` PostgreSQL table. Wire verdict writes and Nexus tag updates.
25. Configure Alertmanager rules for MalwareBazaar hits and OCSP revocation events (immediate auto-block) and VT/YARA hits (flagged for human review).
26. Set recheck schedules: nightly for Tier 1 and developer toolchain artifacts; weekly for all others.

**Phase 6 — AI/ML model artifact pipeline (weeks 23–26)**
27. Deploy `huggingface_hub` (or equivalent hub client) integration for revision resolution and pinning at Stage 4c.
28. Deploy `safetensors` validator and `fickling`/`picklescan` for the legacy-checkpoint exception path.
29. Write the Stage 5c model card/license/lineage capture script and wire it to Nexus tags and the CMDB (or MLflow, per the "Open issues" decision).
30. Stand up the no-egress load-test sandbox (`firejail` or gVisor) for Stage 6 model detonation.
31. Add the model hub-revision-drift check to the Stage 11b recheck job.
32. Populate the pre-approved model license allowlist for the Stage 5c compliance check.

**Phase 7 — Cross-platform authentication, trust root, and remediation (weeks 27–30)**
33. *(Optional — build only if a macOS artifact or endpoint population is confirmed in scope; not required for this estate)* Deploy `codesign`/`spctl` verification on a macOS build agent for Path B macOS artifacts.
34. Deploy `gpg`/`rpm --checksig`/`dpkg-sig` verification and the publisher key register for Path B Linux artifacts — baseline, given the mixed Windows/Linux server estate and Linux developer IDEs.
35. Pin all scanner/sandbox tool versions in the base pipeline image; wire cosign signature verification into the tool-update job.
36. Stand up the canary self-audit job with an initial known-good/known-bad reference sample set.
37. Implement the Stage 11b rate-limit budgeting logic and the `fp_suppressions` table.
38. Wire push-remediation integrations (WSUS/Intune/Ansible) into the recall workflow for managed fleets; document the manual-follow-up SLA for unmanaged systems.
39. Route pipeline-admin configuration changes (ruleset promotion, trust register edits, recheck job config) through a required-second-approver GitLab merge request.
40. Configure `pgBackRest`/`restic` backup jobs for Nexus, the CMDB, and the recheck job datastore; run a test restore.

**Phase 8 — Hardening (ongoing)**
41. Tune YARA rules and Grype thresholds to reduce noise.
42. Conduct the first emergency recall drill covering CVE-triggered (11a), binary recheck-triggered (11b), and model hub-drift scenarios, including a push-remediation dry run.
43. Review and tighten Squid ACLs based on access log analysis.
44. Implement MinIO for long-term SBOM, sandbox report, and recheck log archiving.
45. Review CMDB publisher trust register quarterly — update when publishers rotate certificates or keys.
46. Review all CMDB entries quarterly to confirm owner, expiry, and deployment scope are current.
47. Review false-positive suppression counts and canary-audit history monthly, per "Open issues for internal review."

---

## Open issues for internal review

These mirror the open items in the architecture document, stated here in tooling-specific terms so the implementation team can scope Phase 6/7 work against a decided answer rather than guessing. Close these out with a recorded decision before or during Phase 7.

### 1. VirusTotal paid tier — procurement trigger

The rate-limit budgeting logic (Phase 7, item 37) is a scheduling mitigation, not a capacity increase. Recommend setting a concrete inventory-size trigger now (e.g., budget approval requested automatically once Tier 2/3 staleness exceeds a defined threshold, such as 7 days between rechecks) rather than discovering the ceiling in production. Alternative: treat VirusTotal as Tier 1-only from the start and rely on MalwareBazaar (no meaningful rate limit) as the primary Stage 11b signal for Tier 2/3, deferring the procurement conversation entirely.

### 2. Push-remediation coverage boundary

`WSUS`/`Intune`/`Ansible` integrations (Phase 7, item 38) only reach systems already enrolled in one of those tools. Before building the integration, confirm what fraction of the target fleet is actually enrolled — if it's materially less than 100%, the manual-follow-up SLA process needs to be built with equal priority to the automated path, not as an afterthought.

### 3. SoD enforcement — GitLab MR gate vs. native RBAC

The recommended approach (Phase 7, item 39) routes all pipeline-admin changes through a GitLab merge request for a uniform audit trail. This is simpler to build than configuring native RBAC in every tool (Nexus, YARA repo, CMDB), but it only enforces separation for changes that actually go through Git-tracked configuration — a direct database edit to the CMDB trust register, for instance, would bypass it. Decide whether that residual gap needs closing with native RBAC too, or is accepted given the smaller attack surface of direct-access admin accounts.

### 4. MLflow vs. Nexus tags for the model registry

Phase 6 defaults to Nexus tags plus CMDB (item 29) to avoid standing up a new system before Phase 6 even starts. If the organisation is already running or planning a fine-tuning pipeline, building on Nexus tags now creates migration work later when a proper model registry becomes necessary anyway. Worth deciding this before Phase 6 starts rather than after.

### 5. Export control routing

Not currently modeled in any pipeline stage or script in this document. If Legal already has an export-control review process, the cheapest integration is a webhook from Stage 1 intake (GitLab issue creation) into that existing process, rather than building classification logic into this pipeline.

### 6. Alert-tuning cadence tooling

The `fp_suppressions` table (Phase 7, item 37) gives the data needed to track suppression counts as a metric, but no dashboard or review cadence is built for it yet. A small Grafana panel querying `fp_suppressions` and `artifact_recheck_log` by month, reviewed at the same meeting as the quarterly CMDB review, would close this with minimal additional tooling.
