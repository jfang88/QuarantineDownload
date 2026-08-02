# Enterprise Artifact Ingress: Build-versus-Buy Evaluation

## Document control

| Field | Value |
|---|---|
| Document title | Enterprise Artifact Ingress: Build-versus-Buy Evaluation |
| Version | 1.0 |
| Status | Draft for architecture and procurement review |
| Owner | Security Architecture |
| Last updated | 2026-08-02 |
| Related documents | [`package-intake-architecture.md`](package-intake-architecture.md), [`solution-architecture-tooling.md`](solution-architecture-tooling.md), [`adr/README.md`](adr/README.md) |

## Purpose

This document evaluates how an enterprise can implement the controlled software and file ingress requirements described in this repository. It is not a proof-of-concept plan and does not assume that the organisation should build a custom platform.

The decision is a **build-versus-buy-versus-integrate** decision:

- **Build** means assembling open-source and free components and developing the orchestration, evidence, policy, and lifecycle integrations needed to make them operate as one control environment.
- **Buy** means procuring one or more commercial platforms that provide supported repository, file-security, software-assessment, workflow, model-security, and governance capabilities.
- **Hybrid** means buying mature specialist capabilities while building only the enterprise-specific policy, integration, lifecycle reconciliation, and recall layer.

The central question is not “which scanner should be used?” It is:

> How should the enterprise ensure that external packages, installers, firmware, models, and files enter through a governed path, are evaluated with evidence appropriate to their type, are made available only after approval, and can later be suspended or recalled?

## Executive summary

There is no single commercial product that fully implements the complete target architecture across all artifact types and lifecycle stages.

Three mature commercial markets cover substantial portions of the requirement:

1. **Package repository and software-composition platforms** such as JFrog and Sonatype provide package-manager proxying, policy enforcement, quarantine, vulnerability and licence analysis, approved repositories, waiver workflows, and release promotion.
2. **Secure file-ingress and content-inspection platforms** such as OPSWAT MetaDefender provide multi-engine malware scanning, file-type inspection, archive processing, quarantine, sandbox integration, SBOM and vulnerability assessment, and supervised file-release workflows.
3. **Enterprise software-onboarding platforms and integrations** such as ServiceNow with ReversingLabs Spectra Assure provide self-service software requests, business approvals, final-binary assessment, Pass/Fail policy decisions, audit evidence, CMDB linkage, and remediation workflow.

AI/ML model security is becoming a separate commercial category. Palo Alto Prisma AIRS AI Model Security and Sonatype Firewall now provide model-specific controls, but model governance still needs to be joined to the wider request, evidence, repository, deployment, and recall lifecycle.

The recommended enterprise direction is therefore **hybrid**:

- buy repository, binary-analysis, secure-file-ingress, workflow, and model-security capabilities where they are mature;
- retain a thin enterprise-owned control plane for canonical artifact identity, lifecycle state, cross-product evidence references, expiry, suspension, recall, and deployment-impact reconciliation;
- avoid recreating commercial malware intelligence, SCA, sandboxing, repository, or ITSM products unless a regulatory, sovereignty, air-gap, or cost constraint makes that necessary.

## 1. Why an enterprise ingress capability is needed

External software and files enter an enterprise through many paths:

- package managers such as npm, Maven, PyPI, NuGet, Go, Cargo, APT, Yum, and container registries;
- direct vendor downloads of installers, drivers, firmware, utilities, and patches;
- developer downloads from GitHub releases or arbitrary websites;
- AI/ML models from Hugging Face and other model hubs;
- email, browser, managed file transfer, removable media, partner exchanges, and cross-domain transfer;
- automated CI/CD, agentic tooling, and infrastructure scripts.

Without a controlled ingress model, the organisation cannot reliably answer:

- What exact bytes entered the environment?
- Who requested them and for what purpose?
- What source, publisher, version, hash, signature, and licence were evaluated?
- Which controls passed, failed, were unavailable, or were overridden?
- Who approved use, for which environment, and until what date?
- Where is the approved artifact stored and how may it be consumed?
- Which systems or projects use it?
- What happens when new intelligence later shows that it is unsafe?

Microsoft's Secure Supply Chain Consumption Framework describes the need to control artifact inputs and establish ingestion, scanning, inventory, update, audit, enforcement, rebuild, and remediation practices. OWASP SCVS similarly treats inventory, SBOM, package management, component analysis, and provenance as core assurance families. NIST SP 800-161 and SP 1326 broaden the concern from artifact scanning to supplier, product, provenance, resilience, foundational security, ownership, and supply-chain-tier due diligence.

The enterprise requirement is therefore broader than antivirus scanning or an artifact repository. It is a **governed acquisition and continuing-assurance process**.

## 2. Scope: three related but different ingress problems

A product evaluation must distinguish three capability domains.

### 2.1 Package-manager ingress

This covers dependencies and artifacts resolved through package or container protocols.

Examples:

- npm, PyPI, Maven, NuGet, Go, Cargo;
- APT, Yum/RPM;
- Docker and OCI registries;
- Helm, Conan, Conda, Composer and similar ecosystems.

Key requirement: developers and build systems use an internal proxy or virtual repository instead of reaching public registries directly.

### 2.2 General file and software ingress

This covers arbitrary files and software that are not naturally acquired through a package-manager protocol.

Examples:

- EXE, MSI, DLL, ZIP, TAR, ISO, CAB and shell installers;
- firmware, drivers, appliances and vendor bundles;
- Office documents, PDFs, archives and partner-delivered files;
- patches and utilities downloaded from vendor portals;
- files entering segmented, operational technology, or air-gapped environments.

Key requirement: files are transferred through an inspection and quarantine workflow before release to a destination.

### 2.3 Business software onboarding

This covers the business process surrounding a software request.

Examples:

- an employee requests a new desktop utility;
- a project requests a third-party library or commercial product;
- architecture, security, legal, procurement and software-asset-management reviews are required;
- approved software must be linked to ownership, licence, CMDB, deployment and renewal records.

Key requirement: technical assessment is joined to accountable business approval and operational fulfilment.

A single enterprise solution normally requires products from more than one of these domains.

## 3. High-level requirements

### 3.1 Governance, identity and accountability

The solution should:

- authenticate human and non-human requestors through enterprise identity;
- enforce role-based access and separation of duties;
- prevent requestors from approving their own production use;
- distinguish approval to acquire, approval to promote, and approval to deploy;
- capture business justification, owner, target environment, risk tier, expiry, and review date;
- support legal, procurement, export-control, privacy, and supplier-risk routing where required;
- record exceptions and waivers with approver, justification, scope, and expiry;
- maintain tamper-evident or strongly auditable decision history.

### 3.2 Controlled acquisition

The solution should:

- prevent endpoints and pipelines from retrieving external artifacts through uncontrolled paths;
- support package-proxy protocols and direct file acquisition;
- allow only approved source domains, registries, namespaces, publishers, and paths;
- defend server-side download functions against SSRF, redirects to private addresses, DNS rebinding, metadata-service access, and excessive downloads;
- stream the exact acquired bytes into quarantine while calculating canonical hashes;
- preserve source URL, DNS, TLS, redirect, timestamp, response-header, and byte-count evidence;
- avoid re-downloading different bytes at promotion time.

### 3.3 Integrity, authenticity and provenance

The solution should support evidence appropriate to each artifact type:

- expected and actual hashes;
- package-manager and repository metadata signatures;
- Authenticode, catalog signatures, RPM signatures, repository-signing keys, detached signatures, and vendor-specific signing;
- publisher identity and controlled key or certificate rotation;
- provenance and attestations such as SLSA, Sigstore, in-toto, or equivalent;
- immutable source revision or digest pinning;
- vendor advisories, SBOMs, partial BOMs, model cards, lineage, and licences;
- supplier and product due diligence where artifact-level evidence is insufficient.

### 3.4 Artifact-specific analysis

The solution should not assume that one scanner or one sandbox fits every format.

| Artifact class | Required analysis direction |
|---|---|
| Open-source packages and containers | Package reputation, malicious-package intelligence, SBOM, vulnerabilities, licence, secrets, provenance and dependency-confusion controls |
| Proprietary installers and binaries | Hash and publisher validation, final-binary composition and tampering analysis, malware and reputation, platform-appropriate static or dynamic analysis |
| Firmware and drivers | Vendor signature, static extraction, component identification, hardware/platform compatibility, specialised emulation or test hardware where feasible |
| AI/ML models | Immutable revision, content hash, format and deserialization safety, licence, publisher, model metadata, lineage, backdoor and model-specific analysis |
| Documents and ordinary files | File-type validation, archive recursion, multi-engine malware inspection, DLP and selective content reconstruction |
| Non-executable data and archives | Structural validation, size and decompression limits, content policy and parser isolation |

Every mandatory analysis capability should return an explicit outcome such as `PASS`, `FAIL`, `INCONCLUSIVE`, or `UNAVAILABLE`. The policy must define when a missing or inconclusive result blocks promotion.

### 3.5 Quarantine, evidence and promotion

The solution should:

- keep unapproved artifacts inaccessible to normal consumers;
- preserve rejected and recalled artifacts for investigation without making them consumable;
- link every decision and report to the exact SHA-256 or equivalent immutable identity;
- retain raw reports, normalized verdicts, scanner versions, policy versions, ruleset versions and evidence-bundle references;
- enforce promotion gates rather than relying on advisory approvals;
- promote the already-assessed bytes into approved distribution;
- sign or otherwise protect evidence and promotion records;
- support time-limited exceptions and renewal.

### 3.6 Approved distribution and consumption

The solution should:

- expose package-manager-compatible internal endpoints for developer and CI/CD use;
- expose controlled file-download or deployment channels for generic binaries;
- support repository allowlists, blocklists, namespaces and approved versions;
- enforce internal-only consumption where policy requires it;
- integrate with endpoint management, software distribution, container platforms, configuration management, and model-serving platforms;
- prevent direct access to quarantine storage.

### 3.7 Continuous re-evaluation and recall

The solution should:

- continuously reassess SBOMs and package risk as intelligence changes;
- re-evaluate stored files or hashes against updated malware, tampering, YARA, reputation, publisher and revocation intelligence;
- re-scan or re-assess models when model-security rules or intelligence change;
- suspend new consumption while a high-confidence finding is investigated;
- recall confirmed compromised artifacts;
- identify projects, releases, endpoints, workloads or models affected;
- trigger automated remediation where a management channel exists;
- track manual remediation through an SLA where automation cannot reach;
- retain a complete re-evaluation and disposition history.

### 3.8 Enterprise non-functional requirements

Product selection must also address:

- high availability and disaster recovery;
- data residency and sovereignty;
- air-gapped and disconnected operation;
- throughput, maximum artifact size, nested archives and large models;
- integration APIs, webhooks and event delivery;
- enterprise SSO, MFA, service identities and granular RBAC;
- encryption and key management;
- operational monitoring and audit export;
- supported upgrade and rollback;
- vendor support and incident response;
- retention and legal hold;
- licence terms for automated threat-intelligence use;
- predictable cost at enterprise inventory and scan volume.

## 4. Principal risks to address

| Risk | Why it matters | Required response |
|---|---|---|
| Uncontrolled internet acquisition | Direct downloads bypass approval, evidence and inventory | Force package and file ingress through governed channels |
| Malicious or tampered artifacts | Correct names and versions can still contain injected payloads | Analyse exact final bytes, not only metadata or source |
| Dependency confusion and typosquatting | Public packages may impersonate internal names or trusted projects | Namespace controls, curated proxying, provenance and policy |
| Vulnerable or unsupported components | Known weaknesses may be introduced or discovered later | SBOM, SCA, lifecycle monitoring and update process |
| Signed but compromised software | A valid signature does not guarantee benign content | Combine signatures with binary analysis, reputation and supplier evidence |
| Incomplete proprietary-software visibility | Source-based SCA may not see compiled contents | Final-binary composition and tampering analysis; accept vendor SBOMs |
| Unsafe AI model formats or backdoors | Model files can execute code during load or contain malicious behaviour | Model-specific format, provenance, licence, backdoor and deserialization controls |
| SSRF and downloader abuse | A privileged fetcher can be used to reach internal services | Harden redirects, DNS, destination IPs, TLS, sizes and protocols |
| False assurance from a clean scan | Absence of a detection is not proof of safety | Evidence confidence, defence in depth, explicit residual risk |
| Fragmented evidence | Different tools may disagree on state and identity | Canonical artifact identity and reconciled lifecycle state |
| Waiver and exception drift | Permanent exceptions silently become policy bypasses | Named approval, narrow scope, expiry, renewal and review |
| Intelligence privacy leakage | Hash or file submission can reveal software use or sensitive content | Provider and data-sharing policy; on-premises options for sensitive artifacts |
| Inability to recall deployed software | Blocking repository access does not remove installed copies | Deployment inventory and remediation integration |
| Scanner and platform compromise | The ingress controls are themselves supply-chain dependencies | Pinning, signed updates, isolation, canaries, backup and rollback |
| Excessive custom engineering | A security platform can become unmaintainable and under-supported | Buy mature specialist functions and minimise custom code |
| Commercial lock-in and cost escalation | Deep integration may constrain future choices | Standards-based evidence, portable hashes/SBOMs, API and exit requirements |

## 5. The “why” and the alternative “how” approaches

The architecture should remain stable even when the product choices change.

### Why

The enterprise must establish a single accountable control model that:

1. controls external artifact inputs;
2. verifies exact bytes, source, publisher and provenance;
3. applies analysis appropriate to the artifact type;
4. prevents use until required evidence and approvals exist;
5. distributes only approved artifacts;
6. continuously reassesses approved inventory;
7. suspends, recalls and remediates when new risk is discovered.

### How

There are three viable implementation approaches.

#### Approach A: build with open-source and free components

Assemble specialist components and build the integration layer.

Typical components include:

- Keycloak for identity;
- GitLab Free/CE, OpenProject, or a custom portal for requests;
- Squid or a hardened custom fetch service;
- PostgreSQL for workflow, evidence and lifecycle state;
- a repository manager or object store for quarantine and approved storage;
- Syft, Grype and OWASP Dependency-Track for SBOM and SCA;
- ClamAV, YARA and local reputation datasets;
- CAPE, disposable VMs, gVisor, Kata, binwalk or other format-specific analysis;
- MLflow and model-format scanners for model metadata and safety;
- OpenSearch, Prometheus and Grafana for operations;
- custom services for promotion, attestations, re-evaluation, recall and reconciliation.

This approach is not “free” in an enterprise sense. Licence cost may be lower, but engineering, operations, security hardening, integration testing, rule maintenance, false-positive handling, upgrades, backup, incident response and support remain organisational costs.

#### Approach B: buy commercial platforms

Procure specialist products that provide supported controls and enterprise integrations.

A commercial solution normally combines:

- ITSM and request workflow;
- package repository and curation;
- final-binary or file security;
- model security where required;
- CMDB and software asset management;
- endpoint or workload remediation.

The enterprise buys product capability and support but still performs architecture, policy, integration, data governance and operating-process design.

#### Approach C: hybrid

Buy mature specialist capabilities and build a thin control plane.

The custom layer owns:

- canonical artifact identity;
- cross-product request and lifecycle state;
- evidence references and reconciliation;
- enterprise-specific approval policy;
- expiry, suspension and recall;
- deployment-impact queries;
- reporting across products.

The commercial products own:

- package protocols and repository distribution;
- malware and binary intelligence;
- SBOM and SCA;
- secure file transfer and inspection;
- model analysis;
- ITSM workflow and CMDB;
- endpoint deployment and remediation.

This approach normally provides the best balance for a medium or large enterprise.

## 6. Open-source/free versus commercial comparison

| Dimension | Open-source/free build | Commercial buy | Hybrid |
|---|---|---|---|
| Up-front licence cost | Low to moderate | High | Moderate to high |
| Engineering effort | Very high | Moderate | Moderate |
| Time to initial capability | Slow | Fast for product-covered paths | Medium |
| Coverage breadth | Potentially broad, but assembled manually | Strong within each vendor's product domain | Broadest practical coverage |
| Deep threat intelligence | Limited unless licensed separately | Stronger commercial datasets and research | Commercial engines with enterprise policy |
| Package protocol support | Requires selecting and operating repository tools | Mature and supported | Commercial repository recommended |
| Generic file ingress | Requires custom portal and scanner orchestration | Mature MFT/file-security products available | Commercial gateway recommended where in scope |
| Proprietary final-binary analysis | Difficult to reproduce comprehensively | Strong specialist products available | Buy specialist assessment |
| AI model security | Emerging open-source toolchain; integration heavy | Commercial category is developing rapidly | Buy where material; retain provenance controls |
| Approval and ITSM integration | Custom development or community workflow products | Mature workflow, SLA and CMDB capabilities | Existing ITSM as system of engagement |
| Evidence and attestation | Can be standards-based but must be engineered | Product-native signed evidence varies by vendor | Normalize key evidence in enterprise control plane |
| Continuous re-evaluation | Custom scheduling, feeds and disposition | Strong within vendor datasets; cross-product gap remains | Vendor reassessment plus enterprise recall workflow |
| Air-gap and sovereignty | Strong when self-hosted and carefully designed | Product-dependent; may require on-premises editions | Select on-premises products for sensitive paths |
| Vendor support | Community or self-support | Contractual support and escalation | Support for critical engines; internal support for integration |
| Upgrade and compatibility risk | Owned by the enterprise | Shared with vendors | Reduced but not eliminated |
| False-positive operations | Entirely enterprise-owned | Vendor tooling helps but policy remains enterprise-owned | Shared |
| Lock-in | Lower at component level, higher in custom code | Higher product and data-model lock-in | Manage through standards and portable evidence |
| Total cost of ownership | Often underestimated | More visible but potentially expensive | Usually the most controllable enterprise outcome |

## 7. Commercial capability landscape

### 7.1 JFrog Platform

Relevant capabilities include Artifactory, Xray, Curation, AppTrust, Evidence and Release Lifecycle Management.

Strengths:

- package and container proxying and approved internal repositories;
- policy evaluation before package download or caching;
- blocking, dry-run and audit modes;
- manually approved, automatically approved, or forbidden waiver requests;
- time-limited waivers and decision-owner groups;
- policy enforcement against cached packages;
- vulnerability and licence-based promotion gates;
- signed DSSE/in-toto-style evidence attached to artifacts, builds, packages and release bundles;
- immutable, signed Release Bundles and promotion evidence recording who, when and where;
- strong fit for developer and CI/CD package ingress.

Gaps:

- not a complete business software-request or supplier-due-diligence platform;
- general arbitrary-file transfer and multi-engine file inspection are not its primary design centre;
- deep proprietary final-binary assurance may require ReversingLabs or another specialist;
- endpoint recall and remediation require external inventory and management tools;
- the enterprise-wide lifecycle still needs configuration or an external orchestration layer.

Best fit: development-heavy enterprises seeking a strong package, evidence, promotion and distribution platform.

### 7.2 Sonatype Platform

Relevant capabilities include Nexus Repository, Repository Firewall, Lifecycle, IQ Server and SBOM capabilities.

Strengths:

- policy evaluation and quarantine at the proxy stage;
- prevention of downloads for quarantined components;
- waiver and re-evaluation workflows;
- broad package ecosystem coverage;
- support for Docker, Hugging Face model extensions and raw static-site proxying;
- component intelligence, malicious-package protection, SCA and licence policy;
- strong fit for preventing new open-source and model risk entering proxy repositories.

Gaps and caveats:

- it remains primarily a component and package-consumption firewall rather than a general enterprise file-ingress gateway;
- quarantine semantics focus on newly requested components; a waiver expiry does not necessarily re-quarantine an already cached component without additional action;
- arbitrary vendor binaries and firmware still require evidence, assessment and workflow outside the package proxy;
- business approval, CMDB linkage, deployment inventory and recall require integration.

Best fit: enterprises already standardised on Nexus/Sonatype and prioritising package-consumption control.

### 7.3 OPSWAT MetaDefender MFT and MetaDefender Core

Strengths:

- supervised file-release workflows with approval, revocation, comments and approval history;
- support for multiple distinct supervisors in multi-stage approval;
- multi-engine malware scanning, archive recursion, file-type analysis, DLP and sandbox options;
- quarantine with manual or automatic deeper analysis;
- SBOM and file-based vulnerability-assessment capabilities;
- APIs for streaming file submission, hash lookup, quarantine retrieval, report export and SBOM reports;
- strong deployment fit for regulated, segmented, critical-infrastructure and cross-domain file flows.

Gaps and caveats:

- it is not a package-manager-compatible enterprise repository;
- software packages still need promotion into Artifactory, Nexus or a deployment system;
- content disarm and reconstruction must be used selectively because modifying signed software invalidates its hash and signature;
- business procurement, licence, CMDB, deployment and recall processes still need integration.

Best fit: arbitrary file and software ingress, managed file transfer, IT/OT exchange, air-gap and high-assurance transfer scenarios.

### 7.4 ReversingLabs Spectra Assure

Strengths:

- analyses the final published binary without requiring source code;
- accepts software directly from a download URL before it enters the corporate environment;
- detects malware, tampering, vulnerabilities, secrets and suspicious composition;
- produces policy-based Pass/Fail outcomes and portable assessment evidence;
- supports version-to-version differential analysis and continuing supplier accountability;
- integrates with ServiceNow for self-service requests, assessment, CMDB linkage and remediation tasks;
- well suited to commercial software, third-party packages and complex installers.

Gaps:

- not a package repository or transparent package-manager proxy;
- does not replace ServiceNow or another request and approval system;
- approved artifacts still need a controlled distribution repository or endpoint deployment channel;
- enterprise lifecycle and recall across all artifact classes remain integration responsibilities.

Best fit: employee-requested commercial software, third-party product onboarding, procurement assurance and final-binary risk assessment.

### 7.5 ServiceNow

Strengths:

- Service Catalog, request fulfilment and Workflow Studio;
- multi-step approvals, tasks, SLAs, notifications and rejection paths;
- CMDB, software asset management, procurement and third-party-risk integration;
- strong system of engagement for business request, ownership and operational fulfilment;
- IntegrationHub supports orchestration with scanning and repository products.

Gaps:

- it is not itself a malware, package, binary, firmware or model analysis engine;
- it is not the package or file distribution plane;
- technical verdict quality depends on integrated specialist products;
- custom workflow design is still required to preserve exact artifact identity and evidence.

Best fit: enterprise request, approval, ownership, CMDB, exception, SLA and remediation orchestration.

### 7.6 Palo Alto Prisma AIRS AI Model Security

Strengths:

- model scanning from Hugging Face, local storage, S3, Azure, Google Cloud, JFrog Artifactory and GitLab Model Registry;
- blocking and non-blocking rules;
- checks for deserialization threats, insecure formats, licence, publisher verification, blocklists and model backdoors;
- content-based model fingerprinting and version tracking;
- audit trail and API/SDK integration.

Gaps:

- it covers model-security decisions, not the complete enterprise artifact lifecycle;
- private source, credential and data-governance constraints require design review;
- model approval, repository promotion, deployment inventory and recall still require surrounding platforms.

Best fit: enterprises with material AI/ML model intake or internal model governance requirements.

## 8. Commercial solution patterns

### Pattern 1: package-centric enterprise

```text
ServiceNow or existing ITSM
        |
        v
JFrog Curation/Artifactory/Xray/AppTrust
        |
        +--> ReversingLabs for high-risk proprietary binaries
        +--> Prisma AIRS for models
        |
        v
Approved package and model repositories
        |
        v
CI/CD, developer workstations and deployment platforms
```

Use when most inbound artifacts are dependencies, containers, build outputs and models.

### Pattern 2: Nexus/Sonatype enterprise

```text
ServiceNow or existing ITSM
        |
        v
Sonatype Repository Firewall + Nexus + Lifecycle
        |
        +--> ReversingLabs or OPSWAT for generic binaries/files
        +--> specialist model security where required
        |
        v
Approved repositories and deployment channels
```

Use when the enterprise already operates Nexus and wants strong package-consumption controls.

### Pattern 3: software-onboarding enterprise

```text
ServiceNow Service Catalog
        |
        v
ReversingLabs Spectra Assure
        |
        v
Approval and CMDB record
        |
        v
Artifactory/Nexus or endpoint packaging system
        |
        v
Intune / Configuration Manager / Ansible
```

Use when the main problem is employees or business units requesting third-party software and installers.

### Pattern 4: high-assurance file ingress

```text
External source or transfer zone
        |
        v
OPSWAT MetaDefender MFT/Core
        |
        v
Supervisor approval and quarantine
        |
        v
Artifactory/Nexus/object store or controlled destination
```

Use for regulated file transfer, partner exchange, IT/OT boundaries and disconnected environments.

### Pattern 5: recommended unified hybrid

```text
                         +------------------------------+
                         | ServiceNow / enterprise ITSM |
                         | request, approval, CMDB, SLA |
                         +---------------+--------------+
                                         |
              +--------------------------+---------------------------+
              |                          |                           |
              v                          v                           v
     Package-manager path       Generic software/file path       Model path
     JFrog or Sonatype          ReversingLabs or OPSWAT          Prisma AIRS,
              |                          |                        Sonatype or
              +--------------------------+------------------------specialist
                                         |
                                         v
                          Approved repositories and
                          enterprise deployment channels
                                         |
                                         v
                     Enterprise lifecycle and recall integration
```

The enterprise-owned integration layer preserves a consistent lifecycle and evidence model while allowing specialist products to change.

## 9. Capability comparison

Legend:

- **Strong**: product category provides mature native capability.
- **Partial**: supported with configuration or integration.
- **External**: another product or custom service is normally required.

| Capability | Open-source build | JFrog | Sonatype | OPSWAT | ReversingLabs + ServiceNow | Unified hybrid |
|---|---|---|---|---|---|---|
| Business request and approvals | Custom | Partial | Partial | Partial/Strong for file supervision | Strong | Strong |
| Package-manager proxy | Repository-dependent | Strong | Strong | External | External | Strong |
| Pre-download package blocking | Custom intelligence | Strong | Strong | External | External | Strong |
| Arbitrary URL/software assessment | Custom | Partial | Partial via raw proxy | Strong for submitted files | Strong | Strong |
| Multi-engine file scanning | Custom/licensed engines | External | External | Strong | Binary-analysis focused | Strong |
| Final-binary tampering/composition | Difficult | External | External | Partial | Strong | Strong |
| SBOM/SCA | Strong with integration | Strong | Strong | Partial/Strong by licence | Strong final-binary assessment | Strong |
| Model-specific security | Custom/emerging | Partial | Stronger and growing | Partial | Partial | Strong with specialist |
| Quarantine | Custom | Strong for curated packages | Strong for proxied components | Strong | Workflow-dependent | Strong |
| Signed promotion evidence | Custom with in-toto/Sigstore | Strong | Partial | Report/audit focused | Assessment evidence; repository external | Strong |
| Approved repository distribution | Repository-dependent | Strong | Strong | External | External | Strong |
| CMDB and software-asset linkage | Custom | External | External | External | Strong | Strong |
| Continuous reassessment | Custom | Strong within platform | Strong within platform | Re-scan supported | Strong version assessment | Strong |
| Cross-product recall and remediation | Custom | External | External | External | Partial through ServiceNow | Enterprise integration required |
| Air-gap/high-assurance file transfer | Strong if engineered | Product-dependent | Product-dependent | Strong | Product-dependent | Strong with selected products |

## 10. Build-versus-buy decision criteria

### Favour a commercial solution when

- rapid time to value is required;
- commercial software and final binaries are a major part of the scope;
- enterprise threat intelligence and research are required;
- regulated support, SLAs, vendor escalation or certification matter;
- high throughput or large inventories make community integrations expensive;
- ServiceNow, JFrog, Nexus, Intune or another strategic platform is already deployed;
- the organisation cannot staff continuous scanner, feed, sandbox and ruleset operations;
- model security requires specialised research beyond basic format checks;
- the cost of a false negative or delayed recall is high.

### Favour an open-source build when

- the environment must be fully disconnected or sovereign;
- commercial products cannot meet data-residency or classification constraints;
- scope and volume are limited;
- the organisation has strong platform engineering, security research and operational capability;
- the enterprise accepts longer implementation time and owns the resulting product lifecycle;
- requirements are genuinely unusual and cannot be configured in commercial products;
- avoiding vendor dependence is more important than minimising engineering cost.

### Favour a hybrid when

- different artifact classes need different specialist controls;
- the organisation already owns an ITSM, repository or endpoint-management platform;
- no single product covers the complete lifecycle;
- enterprise-specific approval, lifecycle and recall rules must remain portable;
- commercial analysis is desirable but evidence and state must remain under enterprise control;
- product replacement and exit strategy are important.

## 11. Total-cost-of-ownership model

The decision should compare at least five years of cost, not only annual licence fees.

### Open-source build cost categories

- platform and security engineering;
- custom portal, APIs and workflow;
- repository and storage operations;
- scanner and sandbox infrastructure;
- threat-intelligence subscriptions that are still required;
- ruleset and vulnerability-database maintenance;
- false-positive investigation;
- support, on-call and incident response;
- upgrades, compatibility and regression testing;
- backup, DR and restore testing;
- audit evidence and compliance documentation;
- staff continuity and knowledge concentration.

### Commercial cost categories

- subscriptions and consumption-based scanning;
- repository storage and data transfer;
- premium integrations and connectors;
- non-production and DR licensing;
- professional services and implementation;
- custom workflow and API integration;
- vendor training and certification;
- price escalation and product-tier changes;
- exit and migration costs;
- residual internal operations and policy ownership.

### Hybrid cost categories

Hybrid inherits both licence and integration cost, but should deliberately reduce custom scope. Its business case depends on avoiding duplication:

- do not build a second repository;
- do not build proprietary malware intelligence;
- do not build a general ITSM platform;
- do not build model research capability;
- do build only the canonical integration, policy and recall functions that no specialist product owns end to end.

## 12. Procurement and proof requirements

A vendor evaluation should require live evidence for the organisation's actual artifact classes.

### Mandatory evaluation scenarios

1. Block a known-malicious package before it is cached or downloaded.
2. Request and approve a time-limited waiver with separate identities.
3. Revoke or expire the waiver and prove future consumption is blocked.
4. Acquire a direct vendor installer from an approved URL and preserve the exact hash.
5. Detect a deliberately altered signed package or checksum mismatch.
6. Analyse a complex proprietary installer without source code.
7. Store and export machine-readable evidence linked to the exact artifact hash.
8. Promote the same assessed bytes into an approved repository.
9. Reassess an approved artifact after intelligence changes and suspend new consumption.
10. Identify affected projects or deployed systems and create remediation work.
11. Scan a safe-format and an unsafe-format AI model, where models are in scope.
12. Operate through an external-intelligence outage without silently approving.
13. Demonstrate RBAC, separation of duties and denied self-approval.
14. Export audit records into the SIEM.
15. Restore repository, workflow and evidence records from backup.
16. Demonstrate on-premises, air-gap or data-residency behaviour where required.

### Required vendor questions

- Which artifact protocols and file types are natively supported?
- Is policy evaluated before download, before cache, before promotion, or only after storage?
- Can an already cached or approved artifact be immediately blocked again?
- How are waivers scoped, approved, expired and revoked?
- What exact evidence is retained and can it be exported?
- Are promotion records signed or tamper evident?
- What data, hashes or files leave the organisation?
- Which services require cloud connectivity?
- How are confidential or classified artifacts handled?
- Which threat-intelligence rights are included for automated enterprise use?
- Can raw and normalized results be accessed by API?
- What are maximum artifact sizes, archive depths and model sizes?
- How are false positives and suppression changes audited?
- How are old versions, recalled artifacts and legal holds retained?
- How does the product identify deployed or consumed instances?
- What remediation integrations are available?
- What is the product's own secure-update and supply-chain assurance process?
- What happens when the subscription ends?

## 13. Recommended decision

Adopt a **hybrid enterprise architecture** as the working recommendation.

### Buy

- ITSM, request and CMDB capability where the enterprise already has ServiceNow or an equivalent platform;
- a commercial package platform, normally JFrog or Sonatype;
- commercial final-binary analysis for proprietary software where risk and volume justify it, with ReversingLabs as a strong candidate;
- OPSWAT where general file ingress, managed transfer, segmentation or air-gap workflows are material;
- commercial model security where model intake is material;
- endpoint and workload management for deployment inventory and remediation.

### Build or retain enterprise control of

- the canonical artifact identity and cross-product correlation model;
- lifecycle state and allowed transitions;
- policy mapping from business approval to technical gates;
- evidence references and reconciliation;
- enterprise-specific supplier, legal and privacy routing;
- expiry, suspension, recall and remediation coordination;
- control metrics and executive reporting;
- standards-based evidence export and product exit strategy.

### Avoid

- building a new malware-intelligence platform;
- treating one repository product as a complete general file-ingress solution;
- treating secure file transfer as a package repository;
- treating ITSM approval as technical assurance;
- treating a single clean scan as proof of safety;
- creating a large bespoke platform before confirming which commercial controls already exist.

## 14. Suggested decision sequence

1. Confirm the artifact classes and ingress channels in scope.
2. Measure current direct-download, package-manager and software-request volumes.
3. Inventory existing strategic platforms: ServiceNow, JFrog, Nexus, endpoint management, EDR, SIEM and MLOps.
4. Define mandatory controls and evidence independent of products.
5. Shortlist products by the three capability domains: package ingress, file/software ingress, and business onboarding.
6. Run the mandatory evaluation scenarios using real but non-sensitive representative artifacts.
7. Model five-year total cost of ownership and operating staffing.
8. Select the product combination and document the remaining integration gaps.
9. Create ADRs for the repository platform, generic file-ingress product, binary-analysis product, model-security product and canonical lifecycle system.
10. Implement the smallest integration layer that preserves enterprise lifecycle and recall requirements.

## 15. References

### Standards and guidance

- [Microsoft Secure Supply Chain Consumption Framework](https://www.microsoft.com/en-us/securityengineering/sdl/s2c2f)
- [OWASP Software Component Verification Standard](https://scvs.owasp.org/)
- [NIST SP 800-161 Rev. 1 — Cybersecurity Supply Chain Risk Management Practices](https://csrc.nist.gov/pubs/sp/800/161/r1/upd1/final)
- [NIST SP 1326 — Cybersecurity Supply Chain Risk Management Due Diligence](https://csrc.nist.gov/pubs/sp/1326/final)

### Commercial product documentation

- [JFrog Curation rollout overview](https://docs.jfrog.com/security/docs/curation-rollout-overview)
- [JFrog Curation waiver management](https://docs.jfrog.com/security/docs/manage-waivers)
- [JFrog Evidence](https://docs.jfrog.com/governance/docs/evidence-quick-start)
- [JFrog Release Bundle promotion evidence](https://docs.jfrog.com/governance/docs/promote-a-release-bundle-v2-version)
- [Sonatype Repository Firewall](https://help.sonatype.com/en/repository-firewall.html)
- [Sonatype Firewall quarantine](https://help.sonatype.com/en/firewall-quarantine.html)
- [OPSWAT MetaDefender Core](https://www.opswat.com/products/metadefender/core)
- [OPSWAT MetaDefender Core API](https://www.opswat.com/docs/mdcore/metadefender-core/ref)
- [OPSWAT MetaDefender Managed File Transfer approval](https://www.opswat.com/docs/mdmft/operating/pending-approval-page)
- [ReversingLabs secure software onboarding](https://www.reversinglabs.com/solutions/secure-software-onboarding)
- [ReversingLabs integration with ServiceNow ITSM](https://www.reversinglabs.com/servicenow-integration-for-secure-itsm)
- [ServiceNow Service Catalog request fulfilment](https://www.servicenow.com/docs/r/servicenow-platform/service-catalog/request-fulfillment.html)
- [Palo Alto Prisma AIRS AI Model Security](https://docs.paloaltonetworks.com/ai-runtime-security/ai-model-security/model-security-to-secure-your-ai-models)
- [Palo Alto AI Model Security supported sources and scanning](https://docs.paloaltonetworks.com/ai-runtime-security/ai-model-security/model-security-to-secure-your-ai-models/get-started-with-ai-model-security/scanning-models)
