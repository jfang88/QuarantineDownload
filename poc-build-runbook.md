# Package Intake POC Build Runbook and Demo Script

## Purpose

This runbook describes how to build, operate, demonstrate, reset, and shut down the package-intake POC defined in [`poc-deployment-plan.md`](poc-deployment-plan.md).

It is written as an implementation guide rather than a production operating procedure. Replace example secrets, hostnames, images, and fixture hashes before use. Pin all container images and downloaded tools to tested immutable digests in the actual implementation.

## Target outcome

At the end of the build, the operator can demonstrate:

- Keycloak login for a requestor and an independent approver;
- a request moving through approval, controlled download, quarantine, scanning, cooling, promotion, and approved consumption;
- self-approval being denied;
- checksum, EICAR, unsafe-model, and SSRF cases being blocked;
- a synthetic post-approval compromise signal suspending and recalling an artifact;
- complete audit and evidence records for each action.

## Reference implementation layout

The POC repository implementation can use the following structure:

```text
poc/
├── README.md
├── compose.yaml
├── .env.example
├── config/
│   ├── keycloak/
│   │   └── realm-export.json
│   ├── clamav/
│   ├── yara/
│   │   ├── baseline/
│   │   └── recheck/
│   └── fetch-policy/
│       ├── allowlist.yaml
│       └── blocked-networks.yaml
├── portal/
│   ├── Dockerfile
│   ├── app/
│   ├── migrations/
│   └── tests/
├── worker/
│   ├── Dockerfile
│   ├── intake_worker/
│   ├── recheck_worker/
│   └── tests/
├── mock-source/
│   ├── nginx.conf
│   └── fixtures/
│       ├── good/
│       ├── bad-checksum/
│       ├── eicar/
│       ├── model-safe/
│       └── model-unsafe/
├── scripts/
│   ├── bootstrap.sh
│   ├── healthcheck.sh
│   ├── seed-demo-data.sh
│   ├── run-recheck.sh
│   ├── reset-demo.sh
│   └── export-evidence.sh
├── schemas/
│   ├── request.schema.json
│   ├── evidence-bundle.schema.json
│   └── synthetic-ioc.schema.json
└── docs/
    ├── demo-cheat-sheet.md
    └── screenshots/
```

The architecture documents can remain at the repository root. The POC code may be added under `poc/` in a later implementation change.

## Prerequisites

### Host

Recommended host for the standard demo profile:

- Linux host or Linux VM;
- 10-12 vCPU;
- 24-32 GB RAM;
- 150-250 GB free SSD storage;
- Docker Engine with Compose v2, or Podman with Compose compatibility;
- `curl`, `jq`, `openssl`, `sha256sum`, and `make` or equivalent;
- browser access to the POC host;
- time synchronization enabled.

A smaller host can run the core profile if Dependency-Track, Grafana, and all sandbox services are disabled.

### DNS and hostnames

Use a local DNS zone or host-file entries such as:

```text
127.0.0.1 idp.poc.local
127.0.0.1 portal.poc.local
127.0.0.1 source.poc.local
127.0.0.1 objects.poc.local
127.0.0.1 grafana.poc.local
```

When the stack runs on a remote VM, replace `127.0.0.1` with the VM address. Use a private CA or a local-development certificate issuer if HTTPS is required for the demonstration.

### Secrets

Create local secrets that are not committed:

- PostgreSQL administrator and application passwords;
- Keycloak administrator password;
- portal OIDC client secret;
- object-store root and application credentials;
- worker service credential;
- evidence-bundle signing key if enabled.

The `.env.example` file should contain names and placeholders only.

## Suggested Compose services

| Service | Profile | Purpose |
|---|---|---|
| `reverse-proxy` | `core` | TLS termination and routing. |
| `postgres` | `core` | Portal, workflow, audit, and evidence data. |
| `keycloak` | `core` | OIDC identity provider. |
| `portal` | `core` | Request, approval, evidence, promotion, and consumption UI/API. |
| `object-store` | `core` | Quarantine, approved, evidence, and raw-report buckets. |
| `mock-source` | `core` | Deterministic local source for fixtures. |
| `fetch-worker` | `scan` | Controlled download and acquisition evidence. |
| `clamav` | `scan` | Anti-malware test scanning. |
| `analysis-worker` | `scan` | File type, checksum, YARA, Syft, Grype, and model checks. |
| `recheck-worker` | `scan` | Scheduled or manually triggered approved-inventory recheck. |
| `dependency-track` | `sca` | Optional SBOM ingestion and re-evaluation. |
| `dependency-track-db` | `sca` | Optional Dependency-Track database. |
| `prometheus` | `observe` | Optional metrics. |
| `grafana` | `observe` | Optional dashboards. |
| `openldap` | `directory` | Optional directory federation demonstration. |

## Network configuration

Create separate networks and make the control network internal.

```yaml
networks:
  front:
  control:
    internal: true
  fetch:
  management:
    internal: true
```

Recommended membership:

- `reverse-proxy`: `front`, `control`
- `portal`: `control`
- `keycloak`: `control`
- `postgres`: `control`
- `object-store`: `control`
- `fetch-worker`: `control`, `fetch`
- `mock-source`: `fetch`
- `analysis-worker`, `clamav`, `recheck-worker`: `control`

The mock source allows the complete demo without external access. When live external sources are enabled, only `fetch-worker` receives that route. Enforce the same rule on the host firewall; Compose network names alone are not a production egress control.

## Environment configuration

Example `.env` values:

```dotenv
POC_DOMAIN=poc.local
POSTGRES_DB=package_intake
POSTGRES_USER=package_intake
POSTGRES_PASSWORD=replace-me
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=replace-me
OIDC_REALM=package-intake-poc
OIDC_CLIENT_ID=package-intake-portal
OIDC_CLIENT_SECRET=replace-me
OBJECT_STORE_ACCESS_KEY=poc-application
OBJECT_STORE_SECRET_KEY=replace-me
QUARANTINE_BUCKET=quarantine
APPROVED_BUCKET=approved
EVIDENCE_BUCKET=evidence
REPORTS_BUCKET=raw-reports
DEMO_COOLING_SECONDS=60
MAX_DOWNLOAD_BYTES=1073741824
MAX_REDIRECTS=3
ALLOW_EXTERNAL_FETCH=false
```

Use a development-only cooling period of 60 seconds or a controlled `advance demo clock` action. Never reuse this setting as a production policy.

## Build procedure

### Step 1: create the project skeleton

```bash
mkdir -p poc/{config/{keycloak,yara/baseline,yara/recheck,fetch-policy},portal,worker,mock-source/fixtures,scripts,schemas,docs}
cd poc
cp .env.example .env
chmod 600 .env
```

Populate `.env` with non-default secrets.

### Step 2: create fixture files

Use harmless, deterministic fixtures.

#### Good open-source fixture

```bash
mkdir -p mock-source/fixtures/good/package-1.0
printf 'demo package version 1.0\nunique-marker: PACKAGE-INTAKE-GOOD\n' \
  > mock-source/fixtures/good/package-1.0/README.txt

tar -C mock-source/fixtures/good -czf \
  mock-source/fixtures/good/package-1.0.tar.gz package-1.0

sha256sum mock-source/fixtures/good/package-1.0.tar.gz \
  > mock-source/fixtures/good/package-1.0.tar.gz.sha256
```

#### Bad-checksum fixture

Use the same harmless file, but enter a different expected hash in the request template. Do not modify the fixture itself; the demonstration should clearly show a request/evidence mismatch.

#### EICAR fixture

Create the standard EICAR test file from the official test string using an operator-approved method. Store it only in the isolated fixture directory. Label it clearly as an anti-malware test file, not malware.

Expected result: ClamAV detects it and the pipeline rejects it.

#### Safe model fixture

Generate a very small valid safe-format file using a pinned local library or include a purpose-built fixture whose creation process is documented. Add a model card JSON file containing:

```json
{
  "model_name": "poc-tiny-model",
  "source_namespace": "poc/local",
  "requested_reference": "main",
  "resolved_revision": "1111111111111111111111111111111111111111",
  "format": "safetensors",
  "license": "demonstration-only",
  "base_model": null,
  "datasets": []
}
```

#### Unsafe model fixture

Create a harmless pickle containing a simple dictionary and no executable payload. The test proves that the format is denied by policy.

### Step 3: define baseline YARA rules

Create a baseline rule that does not match the good fixture and a later recheck rule that does.

`config/yara/baseline/poc-baseline.yar`:

```yara
rule POC_Disallowed_Example_Marker
{
    meta:
        purpose = "POC baseline demonstration"
    strings:
        $marker = "PACKAGE-INTAKE-DISALLOWED"
    condition:
        $marker
}
```

Keep the compromise-simulation rule outside the active baseline until the recall demonstration.

`config/yara/recheck/poc-recall-simulation.yar.disabled`:

```yara
rule POC_Synthetic_Post_Approval_Compromise
{
    meta:
        purpose = "Safe POC recall simulation"
        severity = "high"
    strings:
        $marker = "PACKAGE-INTAKE-GOOD"
    condition:
        $marker
}
```

### Step 4: define fetch policy

`config/fetch-policy/allowlist.yaml`:

```yaml
allowed_hosts:
  - source.poc.local
allowed_schemes:
  - http
  - https
max_redirects: 3
max_download_bytes: 1073741824
```

`config/fetch-policy/blocked-networks.yaml`:

```yaml
blocked_cidrs:
  - 0.0.0.0/8
  - 10.0.0.0/8
  - 100.64.0.0/10
  - 127.0.0.0/8
  - 169.254.0.0/16
  - 172.16.0.0/12
  - 192.0.0.0/24
  - 192.168.0.0/16
  - 224.0.0.0/4
  - ::1/128
  - fc00::/7
  - fe80::/10
blocked_hostnames:
  - localhost
  - metadata.google.internal
```

For the local mock source, allow the specific container-network address through a separate explicit test exception. Do not weaken the general private-address policy for live external-fetch mode.

### Step 5: implement the portal workflow

Minimum portal endpoints or actions:

| Method/action | Purpose | Required role |
|---|---|---|
| `POST /requests` | Submit request | `requestor` |
| `GET /requests/{id}` | View request and evidence | owner, approver, analyst, auditor |
| `POST /requests/{id}/approve-fetch` | Approve controlled fetch | `approver`; actor must differ from requestor |
| `POST /requests/{id}/reject` | Reject request | `approver` or `security_analyst` |
| `POST /requests/{id}/promote` | Approve promotion | `approver` or `security_analyst`; policy checks required evidence |
| `POST /artifacts/{sha256}/suspend` | Stop new consumption | `security_analyst` |
| `POST /artifacts/{sha256}/recall` | Recall artifact | `security_analyst` |
| `GET /artifacts/{sha256}/download` | Download only when `APPROVED` and unexpired | authenticated consumer |
| `GET /audit` | View audit events | `auditor`, `security_analyst`, `platform_admin` |
| `POST /admin/demo/reset` | Reset POC data | `platform_admin` |

Required server-side checks:

- derive identity from the validated OIDC token;
- do not accept requestor or approver identity as a request-body field;
- reject self-approval;
- reject promotion without mandatory evidence;
- reject downloads for every state except `APPROVED`;
- reject expired artifacts;
- make state transitions transactional;
- emit an audit event for successful and denied actions;
- use an idempotency key for worker callbacks.

### Step 6: implement the worker contract

A job should contain only identifiers and approved policy data, not user-supplied shell commands.

Example approved fetch job:

```json
{
  "job_id": "job-0001",
  "request_id": "REQ-0001",
  "source_url": "http://source.poc.local/good/package-1.0.tar.gz",
  "declared_type": "open_source",
  "expected_sha256": "<known hash>",
  "risk_tier": "tier-3",
  "approved_by": "<OIDC subject>",
  "approved_at": "<timestamp>"
}
```

The fetch worker should:

1. Parse and normalize the URL.
2. Check scheme and hostname against policy.
3. Resolve the hostname.
4. Reject blocked IP ranges.
5. Connect using the validated destination while preserving the hostname for TLS checks when HTTPS is used.
6. Re-run checks for every redirect.
7. Enforce response and byte limits.
8. Stream the response into quarantine while calculating SHA-256.
9. Record DNS, destination, redirect, TLS, header, byte-count, and timestamp evidence.
10. Submit a signed or authenticated callback containing the object version and hash.

The analysis worker should:

1. Read the quarantine object as read-only.
2. Detect file type and compare it with the declared type.
3. Compare expected and actual SHA-256 when an expected hash exists.
4. Run ClamAV and YARA.
5. Run path-specific checks.
6. Store raw reports in the reports bucket.
7. Write structured verdicts to PostgreSQL.
8. Generate the evidence bundle.
9. Move the request to `ANALYSIS_PASSED`, `ANALYSIS_FAILED`, or `INCONCLUSIVE`.

### Step 7: bootstrap PostgreSQL and object storage

Create databases, application roles, and buckets. The object-store application identity should have:

- write/read access to quarantine for workers;
- write/read access to reports and evidence for workers;
- no delete permission during a normal run;
- approved-bucket write permission only for the promotion service identity;
- approved-bucket read permission only through the portal or a constrained consumer identity.

Enable versioning where supported. Object-lock or write-once controls are desirable for the POC if they can be configured reliably, but do not claim immutable retention unless it has actually been tested.

### Step 8: bootstrap Keycloak

Create:

- realm `package-intake-poc`;
- OIDC client `package-intake-portal`;
- service client `package-intake-worker`;
- realm roles `requestor`, `approver`, `security_analyst`, `platform_admin`, `auditor`;
- demonstration users from the deployment plan;
- group-to-role mappings if the directory profile is enabled.

Set temporary passwords and force a password change outside a live scripted demo. For an entirely disposable environment, document that the credentials are demo-only.

### Step 9: start the core profile

```bash
docker compose --profile core up -d
./scripts/healthcheck.sh core
```

Expected health checks:

- PostgreSQL accepts connections;
- Keycloak realm endpoint responds;
- portal health endpoint responds;
- object-store health endpoint responds;
- mock-source good fixture returns the expected byte count.

### Step 10: start the scan profile

```bash
docker compose --profile core --profile scan up -d
./scripts/healthcheck.sh scan
```

Before the demonstration, confirm:

- ClamAV signatures are loaded;
- baseline YARA rules compile;
- Syft and Grype versions/digests are recorded;
- the scanner can read quarantine but cannot reach the internet;
- the fetch worker can reach the mock source;
- the portal cannot reach arbitrary external addresses.

### Step 11: seed demo templates

Create request templates in the portal:

| Template | Declared type | Expected result |
|---|---|---|
| Good package | `open_source` | Approved after scans and promotion. |
| Wrong checksum | `open_source` | Checksum failure. |
| EICAR | `proprietary_binary` or `data` | ClamAV detection and rejection. |
| Safe model | `model` | Safe-format path passes. |
| Unsafe pickle model | `model` | Policy rejection. |
| SSRF loopback | `proprietary_binary` | Fetch policy denial. |
| Scanner unavailable | any | Inconclusive and fail closed. |

## Pre-demo validation checklist

Complete this checklist on the day of the demonstration.

### Environment

- [ ] Host has sufficient free RAM and disk.
- [ ] Core and scan profiles are healthy.
- [ ] System time is correct.
- [ ] Browser can reach Keycloak and portal.
- [ ] Demo user credentials work.
- [ ] PostgreSQL and object-store volumes have been backed up or snapshotted after seeding.

### Fixtures

- [ ] Good fixture hash matches the request template.
- [ ] EICAR fixture is detected by ClamAV.
- [ ] Baseline YARA rules compile and do not match the good fixture.
- [ ] Recall-simulation rule is present but disabled.
- [ ] Safe model fixture passes format validation.
- [ ] Unsafe pickle fixture is rejected.
- [ ] SSRF fixture is blocked.

### Workflow

- [ ] Requestor cannot approve their own request.
- [ ] Approver can approve another user's request.
- [ ] Worker callbacks are authenticated.
- [ ] Promotion is blocked before analysis and cooling complete.
- [ ] Approved download works.
- [ ] Quarantine and rejected download attempts fail.
- [ ] Recall blocks subsequent approved downloads.

### Presentation

- [ ] Browser tabs are prepared for requestor, approver, analyst, and auditor sessions.
- [ ] A terminal is open for health checks and recheck commands.
- [ ] Optional dashboards are open.
- [ ] The environment is reset to the documented starting state.

## Demonstration script

The complete demonstration is approximately 30-45 minutes. A 15-minute version can show sections 1, 2, 3, 6, and 8 only.

### Section 1: explain the topology and identities

**Operator actions**

1. Show the architecture diagram.
2. Show the Keycloak realm and role assignments.
3. Explain that the requestor, approver, analyst, and worker are distinct identities.
4. Show that only the fetch worker has access to the mock source or controlled egress.

**Key message**

The POC is proving a control system, not merely running scanners. Identity, state, evidence, and network boundaries determine whether an artifact may move.

### Section 2: prove self-approval is blocked

**Operator actions**

1. Log in as `requester1`.
2. Submit the `Good package` request.
3. Attempt to call or select `Approve fetch` while still logged in as `requester1`.
4. Show the authorization error.
5. Open the audit view as `auditor1` and show the denied action.

**Expected state:** `REQUESTED`

**Key message:** Separation of duties is enforced by the server, not just by a hidden button.

### Section 3: run the happy path

**Operator actions**

1. Log in as `approver1` in a second browser session.
2. Review the request details and expected hash.
3. Approve the fetch.
4. Show the state moving through `FETCHING`, `ACQUIRED`, and `ANALYSING`.
5. Open acquisition evidence: resolved host and address, redirect chain, byte count, actual SHA-256, and quarantine object version.
6. Open scan evidence: detected file type, checksum match, ClamAV verdict and database version, YARA verdict and ruleset hash, Syft SBOM link, and Grype result.
7. Wait for or advance the demo cooling gate.
8. Approve promotion as `approver1` or `analyst1`, according to the configured policy.
9. Show the state `APPROVED` and the evidence-bundle hash.
10. Download the artifact through the consumer endpoint and verify its SHA-256.

**Expected state:** `APPROVED`

**Key message:** The approved artifact is the same byte sequence that was acquired and analysed, and every decision is linked to evidence.

### Section 4: demonstrate checksum failure

1. Submit the `Wrong checksum` template as `requester1`.
2. Approve fetch as `approver1`.
3. Show the actual file hash and the requested expected hash.
4. Show the failed checksum verdict.
5. Attempt to promote or download the artifact.

**Expected state:** `ANALYSIS_FAILED` or `REJECTED`.

**Key message:** A clean malware scan does not override failed integrity evidence.

### Section 5: demonstrate malware-test detection

1. Submit and approve the EICAR request.
2. Show the ClamAV detection.
3. Show that the object remains in quarantine for evidence but is not available to consumers.
4. Show the raw report and scanner-database version.

**Expected state:** `ANALYSIS_FAILED` or `REJECTED`.

**Key message:** The demonstration uses a standard test signature, not live malware.

### Section 6: demonstrate artifact-specific policy

#### Safe model

1. Submit and approve the safe model fixture.
2. Show format validation, size/resource policy, model card, immutable revision, and ML-BOM record.
3. Show successful no-egress load-test or parser validation.

#### Unsafe pickle model

1. Submit and approve the harmless pickle fixture.
2. Show that the policy rejects the format before any unrestricted load.

**Key message:** Different artifact types follow different evidence paths; the system does not treat a model, Linux package, firmware image, and Windows installer as the same executable.

### Section 7: demonstrate SSRF protection

1. Submit a request with a URL targeting loopback, a private address, or the metadata-service range.
2. Approve the fetch to prove that business approval alone does not bypass fetch controls.
3. Show the resolved destination and blocked policy rule.
4. Optionally demonstrate an allowlisted mock hostname that redirects to a blocked destination.

**Expected state:** `FETCH_FAILED` or `REJECTED`.

**Key message:** The downloader treats user-supplied URLs as hostile input and validates every redirect destination.

### Section 8: demonstrate post-approval compromise and recall

Use the good artifact approved in Section 3.

1. Show that the artifact is currently downloadable.
2. Enable the synthetic compromise rule:

```bash
mv config/yara/recheck/poc-recall-simulation.yar.disabled \
   config/yara/recheck/poc-recall-simulation.yar

docker compose restart recheck-worker
./scripts/run-recheck.sh
```

Alternatively, add the artifact hash to the local synthetic IOC JSON feed.

3. Show the recheck finding, including artifact hash and new ruleset/feed version.
4. Show the automatic transition from `APPROVED` to `SUSPENDED`.
5. Attempt another consumer download and show that it is denied.
6. Log in as `analyst1`, review the evidence, and select `Recall`.
7. Show the `RECALLED` state, owner notification record, and audit trail.
8. Show that the original bytes and evidence remain preserved for investigation.

**Expected state:** `RECALLED`.

**Key message:** Approval is not permanent trust. New information can suspend consumption immediately and trigger a controlled recall.

### Section 9: demonstrate scanner outage and fail-closed behaviour

1. Stop a required scanner:

```bash
docker compose stop clamav
```

2. Submit and approve a new request.
3. Show timeout/retry evidence and the `INCONCLUSIVE` state.
4. Show that promotion is denied.
5. Optionally show a time-limited analyst exception with reason and expiry.
6. Restart the scanner and rerun analysis.

```bash
docker compose start clamav
```

**Key message:** Missing analysis is not silently treated as a pass.

### Section 10: show audit and evidence export

1. Filter the audit view by the original request ID.
2. Show identity, state, reason, timestamp, and correlation ID for each event.
3. Export the evidence bundle:

```bash
./scripts/export-evidence.sh REQ-0001 ./exports/REQ-0001
sha256sum ./exports/REQ-0001/evidence-bundle.json
```

4. Show that the bundle hash matches the database record.

**Key message:** The POC produces evidence that can be reviewed independently of the live user interface.

## Short-form 15-minute demo

1. Show Keycloak users and roles.
2. Submit a good request as requestor.
3. Show failed self-approval.
4. Approve as a separate approver.
5. Show acquisition and scan evidence.
6. Promote and download the approved artifact.
7. Trigger synthetic recheck match.
8. Show suspension, recall, and denied download.
9. Finish on the complete audit timeline.

## Operating procedures

### Start the environment

```bash
cd poc
docker compose --profile core --profile scan up -d
./scripts/healthcheck.sh all
```

Add optional profiles only when needed:

```bash
docker compose --profile core --profile scan --profile sca --profile observe up -d
```

### Stop optional services to save resources

```bash
docker compose --profile observe stop grafana prometheus
docker compose --profile sca stop dependency-track dependency-track-db
docker compose stop recheck-worker analysis-worker clamav fetch-worker
```

Do not use `down -v` unless intentionally resetting all persistent data.

### Restart after a pause

```bash
docker compose --profile core up -d
./scripts/healthcheck.sh core
docker compose --profile scan up -d
./scripts/healthcheck.sh scan
```

Check scanner database freshness before intake.

### Run a manual recheck

```bash
./scripts/run-recheck.sh --scope approved --reason manual-demo
```

Expected output should include total approved artifacts, artifacts scanned, clean/flagged/failed counts, ruleset/feed version, requests suspended, and the job correlation ID.

### Export evidence

```bash
./scripts/export-evidence.sh <request-id> <output-directory>
```

The export should include:

```text
request.json
approvals.json
acquisition.json
scan-summary.json
raw-reports/
evidence-bundle.json
audit-events.json
artifact.sha256
```

### Review failed jobs

1. Open the request and job correlation ID.
2. Check structured worker logs.
3. Confirm whether the failure occurred during fetch, storage, scan, callback, or state transition.
4. Confirm the current lifecycle state.
5. Retry only idempotent jobs.
6. Do not manually copy an object into the approved bucket to bypass the workflow.

### Reset the demonstration

The reset script should disable the synthetic recall rule, restore the seeded database/object-store state, clear queued jobs, reset user sessions if needed, retain application configuration and fixture files, and rerun health and fixture checks.

Example:

```bash
./scripts/reset-demo.sh --confirm RESET-DEMO
./scripts/healthcheck.sh all
```

### Clean shutdown

```bash
docker compose stop recheck-worker analysis-worker clamav fetch-worker
docker compose stop portal keycloak object-store postgres mock-source reverse-proxy
```

Confirm PostgreSQL and object storage completed a clean shutdown before powering off the VM.

## Troubleshooting

### Login loop or invalid redirect URI

- Verify portal URL exactly matches the Keycloak client redirect URI.
- Verify issuer URL, realm name, and browser-resolvable hostname.
- Clear stale browser sessions after changing client configuration.
- Check clock skew between host and containers.

### Request remains in `APPROVED_TO_FETCH`

- Check the queue/job table.
- Confirm fetch worker service identity can authenticate.
- Confirm it can resolve and reach `source.poc.local`.
- Confirm the request URL matches the allowlist.

### Fetch worker rejects the mock source as private

- Keep private-address blocking enabled for external mode.
- Add a specific test-only exception for the exact mock-source service name and network identity.
- Do not add a broad RFC1918 allow rule.

### ClamAV consumes too much memory

- Stop optional services.
- Serialize scans.
- Allocate additional memory to the scanner container.
- Start ClamAV shortly before the demo and stop it afterward.

### Grype or vulnerability database update fails

- Pre-stage the database before switching the analysis network to internal mode.
- Record the database build timestamp.
- Continue the core demo with SCA marked unavailable only if the selected use case does not require it; otherwise fail closed.

### Promotion action is disabled

Check that analysis passed, all mandatory scan records exist, the cooling gate is complete or an authorized demo override exists, the actor has the promotion role, the actor is not prohibited by the configured separation rule, and evidence-bundle generation completed.

### Approved download returns denial

Check lifecycle state, expiry, recall status, object version, and consumer authorization. A denial after the recall demo is expected.

### Recheck does not flag the synthetic artifact

- Confirm the rule file is enabled and compiled.
- Confirm the recheck worker loaded the new ruleset hash.
- Confirm it scanned approved objects, not only new quarantine objects.
- Confirm the good fixture still contains the unique marker.
- Remove any prior false-positive suppression for the same artifact/rule pair.

## Safety rules for demonstrations

- Do not use live malware.
- Do not submit proprietary or confidential artifacts to external analysis services.
- Do not expose the POC directly to the public internet.
- Do not reuse demonstration passwords in another environment.
- Do not enable unrestricted URL fetching.
- Do not mount the container runtime socket into portal or scanner containers.
- Do not run dynamic-analysis guests with access to enterprise credentials or networks.
- Clearly label EICAR, synthetic IOC, and synthetic YARA findings as simulations.
- Preserve evidence of rejected artifacts but restrict access to operators and analysts.

## Build completion checklist

- [ ] Compose files and image digests are recorded.
- [ ] Core services start from a clean host.
- [ ] Keycloak realm, users, roles, and service clients are reproducible.
- [ ] Portal enforces server-side role and self-approval rules.
- [ ] Fetch worker enforces destination and redirect policy.
- [ ] Quarantine and approved storage permissions are distinct.
- [ ] ClamAV, YARA, checksum, and file-type controls work.
- [ ] At least one path-specific open-source or model analysis works.
- [ ] Evidence bundle schema validates.
- [ ] Happy path is repeatable.
- [ ] EICAR and checksum failures are repeatable.
- [ ] SSRF block is repeatable.
- [ ] Scanner outage produces `INCONCLUSIVE` and blocks promotion.
- [ ] Synthetic compromise suspends and recalls an approved artifact.
- [ ] Evidence export matches the database bundle hash.
- [ ] Reset script restores the initial demo state.
- [ ] Optional services can be stopped and restarted without data loss.

## Recommended next implementation changes

After this documentation is approved, implement the POC in small check-ins:

1. `poc/` skeleton, Compose networks, PostgreSQL, Keycloak, and object store.
2. Portal authentication, request schema, roles, lifecycle state machine, and audit events.
3. Controlled fetcher and immutable acquisition evidence.
4. ClamAV, YARA, checksum, file-type, and evidence-bundle pipeline.
5. Promotion and approved-only download controls.
6. Safe fixtures and automated happy/unhappy-path tests.
7. Synthetic recheck, suspension, recall, and reset scripts.
8. Optional Dependency-Track, observability, directory federation, and sandbox profiles.

Each check-in should include automated tests for the transition or control it adds, rather than waiting for an end-to-end demonstration to reveal authorization or lifecycle defects.
