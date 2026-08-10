# Controlled Data Release — Tooling and Implementation Guide

## Document control

| Field | Value |
|---|---|
| Document title | Controlled Data Release — Tooling and Implementation Guide |
| Version | 0.1 |
| Status | Draft for architecture and procurement review |
| Owner | Security Architecture |
| Last updated | 2026-08-10 |
| Related documents | [`controlled-data-release-architecture.md`](controlled-data-release-architecture.md), [`solution-architecture-tooling.md`](solution-architecture-tooling.md), [`enterprise-build-vs-buy-evaluation.md`](enterprise-build-vs-buy-evaluation.md) |

> This is a companion to the existing package-intake tooling and build-versus-buy documents. It introduces the additional capability domain required for controlled operational-data release, cross-domain transfer, and vendor-support upload. It does not select a production product.

---

## 1. Capability model

A production controlled-data-release solution normally spans several product or platform categories. No single category should be assumed to solve the complete problem.

| Capability | What it must provide |
|---|---|
| Request / ITSM workflow | Business justification, source/destination, owner, case reference, risk routing, collection approval, release approval, expiry |
| Identity and privileged access | SSO/MFA, human roles, service identities, least privilege, break-glass controls |
| Controlled collection | Read only approved source scope without granting the requester broad administrative access |
| Release quarantine | Isolated encrypted storage, object immutability/versioning, no normal consumer access |
| File/content inspection | File type, archive recursion, malware, malformed-file handling, parser isolation |
| Secrets detection | Passwords, API keys, tokens, private keys, certificates, cloud credentials, connection strings |
| DLP / classification | PII, customer data, regulated data, confidential data, security-sensitive information |
| Redaction / sanitisation | Mask/remove sensitive content, drop files, minimise structured data, preserve lineage |
| Managed File Transfer / transfer broker | Destination profiles, protocol mediation, audit, receipts, no general routing |
| Policy engine | Risk-based fail-closed decisions, exception handling, destination/classification policy |
| Evidence and lifecycle store | Canonical state, immutable hashes, findings, approvals, transformation lineage, receipts |
| Monitoring / SIEM | Operational health, abuse detection, policy bypass alerts, investigations |
| Encryption / key management | At-rest, in-transit, optional file-level encryption and controlled key exchange |

---

## 2. Fourth enterprise capability domain

The existing build-versus-buy evaluation distinguishes package-manager ingress, general file/software ingress, and business software onboarding. Controlled operational-data release should be treated as a **fourth capability domain**:

### 2.4 Secure operational-data release and cross-domain transfer

This covers data moving from protected enterprise systems to another internal trust zone or external support party.

Examples:

- production logs copied to an operational-assurance/support network;
- diagnostic bundles copied from production to a test or lab network;
- packet captures or crash dumps supplied to specialist support teams;
- redacted database extracts used to reproduce a fault;
- support bundles uploaded to an external vendor portal;
- partner/vendor exchanges where the outbound data must be inspected and destination-bound.

**Key requirement:** the organisation must control collection, inspect and minimise the data, bind release approval to exact bytes and an approved destination, transfer through a non-routable audited broker, and prevent transfer from becoming an administrative change channel.

This domain is not satisfied by a package repository, antivirus scanner, ordinary file share, or ITSM approval workflow alone.

---

## 3. Build, buy, and hybrid patterns

### Pattern A — Build a thin release workflow using existing enterprise platforms

Typical composition:

- existing IdP;
- existing ITSM/workflow;
- hardened collection workers;
- object storage for release quarantine;
- existing malware scanner;
- secrets detection and DLP APIs/engines;
- custom policy/orchestration service;
- managed SFTP/HTTPS connectors;
- evidence database;
- SIEM integration.

**Strengths:** aligns closely to enterprise policy and existing infrastructure; avoids creating another user-facing platform.

**Weaknesses:** integration, evidence correlation, transformation lineage, destination binding, and lifecycle correctness become custom engineering responsibilities.

### Pattern B — Buy Managed File Transfer / secure file exchange and integrate governance

Use a mature MFT/secure exchange platform for transfer, partner profiles, protocol handling, receipts, encryption, and operations; integrate it with enterprise workflow, DLP, secrets inspection, and evidence storage.

**Strengths:** mature protocol/security operations; strong partner/destination administration; supported transfer features.

**Weaknesses:** MFT alone may not provide source-data minimisation, deep DLP/secrets inspection, redaction workflow, or a canonical release lifecycle.

### Pattern C — Buy secure content inspection/DLP plus MFT, retain a thin control plane

This is the likely enterprise hybrid pattern:

- ITSM/identity remain enterprise standards;
- content/DLP/security scanning comes from mature security platforms;
- MFT/cross-domain transfer comes from a supported transfer platform;
- the enterprise owns only the policy bindings, canonical release state, evidence correlation, exception model, and source/destination integrations.

This mirrors the existing repository's broader hybrid direction: buy mature specialist capabilities and avoid rebuilding malware intelligence, file-transfer engines, DLP classification, or repository functions without a strong reason.

---

## 4. Collection tooling requirements

The collection component should be designed separately from the transfer component.

### Required properties

- dedicated service identity;
- source-specific least privilege;
- read-only collection wherever possible;
- explicit allowed path/query/scope;
- maximum bytes/files/records;
- time-window constraints;
- no arbitrary command execution exposed to requesters;
- no use as a general administration proxy;
- hash while collecting;
- write only to release quarantine;
- complete evidence of source scope and collection result.

### Platform patterns

| Source | Preferred collection pattern |
|---|---|
| Linux application/server | Restricted service account, allowlisted paths, agent/API/export job; avoid general root shell |
| Windows server | Restricted service/account access to approved logs/paths; event-log export or application support bundle where feasible |
| Network/security appliance | Vendor/API export, syslog/log archive, restricted support bundle generation |
| Database | Pre-approved read-only diagnostic query/export with field/row limits |
| Kubernetes/container platform | Namespace/workload-scoped log export through platform API/service identity |
| SaaS/cloud service | Provider export/API using scoped service identity and case-specific request |

---

## 5. Release-quarantine storage

Release quarantine is conceptually similar to package quarantine but serves a confidentiality objective.

Required controls:

- encryption at rest;
- object versioning or immutable identity;
- deny ordinary requester/consumer direct access;
- separate original and transformed candidate objects;
- malware/DLP workers receive read access only to required objects;
- transfer worker can read only objects in `APPROVED_FOR_RELEASE` state;
- automatic retention/expiry;
- purge evidence;
- storage keys/object IDs linked to canonical SHA-256.

For a POC, separate S3-compatible buckets or filesystem-backed object stores are sufficient. For production, use the organisation's approved secure object-storage platform or an MFT product's secure staging capability only if it can enforce the same lifecycle/evidence requirements.

---

## 6. Content inspection stack

### 6.1 File and archive inspection

Minimum functions:

- MIME/magic detection;
- extension mismatch;
- archive recursion;
- expansion limits;
- encrypted archive detection;
- malware scan;
- YARA or equivalent pattern scan where useful;
- parser isolation/timeouts.

### 6.2 Secrets detection

A secrets engine should detect, at minimum:

- passwords and common credential assignments;
- cloud access keys;
- API tokens;
- OAuth/JWT/session tokens;
- private keys;
- certificate/private-key bundles;
- database and service connection strings;
- vendor/application-specific secret formats defined by the organisation.

Production selection should prioritise low false-negative risk, support for custom detectors, masking in evidence, and on-prem/private processing for sensitive data.

### 6.3 DLP / data classification

A DLP/classification capability should support policy decisions based on:

- regulated personal data;
- customer identifiers;
- financial/payment data where relevant;
- source code and intellectual property;
- security-sensitive infrastructure information;
- organisation labels/classifications;
- custom dictionaries/patterns;
- destination risk class.

Hash-only public reputation services are not appropriate for DLP because the security question is the content itself, not only whether a file is known-malicious. Sensitive operational files should not be uploaded to public scanning services unless policy and contract explicitly permit it.

---

## 7. Redaction and sanitisation tooling

Different data types require different techniques.

| Data type | Example minimisation approach |
|---|---|
| Text logs | Pattern masking, field removal, selected time range, selected component only |
| JSON/XML | Remove or transform named fields, preserve schema where needed |
| CSV/database extract | Column/row reduction, pseudonymisation, sampling, tokenisation |
| Support archive | Remove unnecessary members, redact text/config members, rebuild archive |
| Packet capture | Capture filters, time/protocol/host reduction; specialised sanitisation where available |
| Memory/core dump | Prefer targeted diagnostic alternatives; if essential, elevated review and specialised tooling |
| Screenshots/documents | Manual or tool-assisted redaction with verification |

The transformation engine must produce a new object and new SHA-256. Never overwrite an already inspected/approved original in place.

---

## 8. Managed File Transfer / transfer broker requirements

The broker is the enforcement point between approved release storage and the destination.

### Mandatory properties

- destination profiles administered separately from requesters;
- no arbitrary user-supplied destination URL at transfer time;
- allowlisted protocol/host/path;
- strong service identity and secret management;
- encryption in transit;
- optional file-level encryption;
- transaction ID/receipt where protocol supports it;
- retry with idempotency;
- maximum transfer size/time;
- no general IP routing;
- no SSH/RDP jump-host role;
- no source-system administrative permissions;
- transfer only when canonical workflow state is `APPROVED_FOR_RELEASE` and hash/destination still match.

### Internal destination examples

- secure object-storage bucket;
- controlled support file share;
- lab/test staging service;
- dedicated troubleshooting enclave.

### External destination examples

- managed vendor SFTP profile;
- vendor HTTPS support upload connector;
- enterprise-managed secure exchange portal;
- partner MFT endpoint.

Where a vendor portal cannot be automated safely, the design may use a controlled human-assisted release step, but the final approved bytes and audit evidence must still remain bound to the case and destination. Browser upload from an unrestricted endpoint should not become the default architecture simply because it is convenient.

---

## 9. Destination-side protections

The destination should receive data into a staging location, not a live execution/configuration path.

Required or strongly preferred:

- non-executable mount/location;
- no automatic post-upload hooks;
- no package install/deploy triggers;
- read access limited to the support/test group;
- separate privileged identity required for any later change;
- normal destination change-management controls remain in force;
- local malware/content controls may re-scan on arrival;
- retention/cleanup policy for the destination copy.

This is the primary technical control preventing the transfer channel from becoming a catastrophic-change mechanism.

---

## 10. Integration with the existing package-intake platform

Reuse is encouraged where the semantics match.

| Existing capability | Reuse for data release? | Notes |
|---|---|---|
| IdP / OIDC | Yes | Same human/service identity foundation |
| Request portal | Yes, with a separate request type/workflow | Do not reuse package states or fields blindly |
| Evidence database | Yes, if schema cleanly separates lifecycle types | Shared audit primitives are useful |
| Object store | Yes, using separate buckets/policies | Release quarantine has different access/retention semantics |
| ClamAV/YARA | Yes | Malware remains relevant for cross-zone files |
| Package SBOM/SCA | Generally no | Not the primary confidentiality control for logs/data |
| Controlled downloader | No | Outbound release uses controlled collector + transfer broker instead |
| Approved repository | No | Use release-approved staging/destination, not a consumable package repository |
| Recheck/recall | Different | Outbound release focuses on cancel-before-transfer, disclosure response, purge, and audit |

---

## 11. Evaluation criteria for procurement

A procurement exercise should score products/integrations against at least:

### Governance

- enterprise SSO/MFA and granular RBAC;
- separation of requester, approver, platform admin;
- API/webhook integration with ITSM/workflow;
- time-limited approvals and exceptions;
- external case/vendor metadata;
- tamper-evident audit.

### Content security

- file-type/archive processing;
- malware scanning;
- custom secrets detectors;
- enterprise DLP/classification integration;
- on-prem/private processing options;
- redaction/sanitisation support;
- handling of encrypted/unsupported content;
- large file, PCAP, dump, and archive limits.

### Transfer security

- fixed destination profiles;
- SFTP/HTTPS/object-store/partner connectors as required;
- no general routing requirement;
- encryption and key management;
- retries/receipts/idempotency;
- destination access controls;
- ability to verify exact transferred hash/object.

### Operations

- HA/DR;
- monitoring and SIEM export;
- retention and purge;
- API quality;
- upgrade/rollback;
- vendor support;
- data residency/sovereignty;
- throughput and maximum file size;
- predictable licensing/cost at expected transfer volume.

---

## 12. Suggested production architecture pattern

```mermaid
flowchart LR
    ITSM[ITSM / Request Portal] --> Policy[Enterprise Policy / Release State]
    IdP[Enterprise IdP] --> ITSM
    Policy --> Collector[Controlled Collectors]
    Collector --> Store[Release Quarantine]
    Store --> Inspect[Private Content Inspection\nMalware + Secrets + DLP]
    Inspect --> Transform[Redaction / Minimisation]
    Transform --> Store2[Final Release Candidate]
    Store2 --> Policy
    Policy -->|approved hash + destination| MFT[MFT / Transfer Broker]
    MFT --> Internal[Internal support / lab]
    MFT --> Vendor[Vendor / partner endpoint]
    Policy --> Evidence[(Evidence DB)]
    Collector --> Evidence
    Inspect --> Evidence
    Transform --> Evidence
    MFT --> Evidence
    Evidence --> SIEM[SIEM / Audit]
```

The preferred enterprise pattern is usually **hybrid**: retain enterprise identity, ITSM, policy, and evidence ownership; buy or reuse mature DLP/content-security and MFT capabilities; build only the orchestration and enterprise-specific control bindings that are not available off the shelf.

---

## 13. POC mapping

The companion POC uses deliberately simple substitutes:

| Production capability | POC substitute |
|---|---|
| Enterprise IdP | Existing Keycloak POC |
| ITSM workflow | Existing purpose-built POC portal |
| Production source collectors | Mock-production file container + restricted worker |
| Secure object storage | Existing S3-compatible store/separate buckets |
| Malware inspection | Existing ClamAV/YARA |
| Secrets/DLP | Deterministic synthetic pattern scanner |
| Redaction | Simple text transformation worker |
| MFT | Small transfer service with fixed destination profiles |
| Vendor portal | Mock HTTP upload endpoint |
| SIEM | Structured audit events / optional existing observability stack |

This keeps the demonstration deterministic and safe while preserving the architectural boundaries that matter.
