# Enterprise Package Intake and Approved Repository — Reference Architecture

> **Status: draft reference architecture and POC design, not a deployed control environment.** This repository documents a proposed design for controlled software acquisition, recommended tooling, review findings, and a proof-of-concept plan. Nothing in this repository should be treated as a production control until it has been implemented, tested, approved, and operated under the organisation's governance processes.

This repository describes a controlled software acquisition architecture for enterprises that need to download packages, binaries, libraries, containers, installers, models, and related artifacts from the internet while reducing supply-chain risk. The target model uses a restricted egress path, request and approval workflow, quarantine repository, integrity and provenance verification, malware screening, cooling-off delay, isolated testing, promotion to a final approved repository, and continuous re-evaluation after approval.

The architecture enforces a single controlled intake path for artifact types used by desktop, server, developer, and AI/ML teams. Endpoints and pipelines consume approved artifacts from internal distribution points rather than retrieving them directly from arbitrary internet sources.

Three analysis paths are defined:

- open-source packages and container images, using SBOM and vulnerability analysis;
- proprietary binaries, installers, firmware, and vendor updates, using vendor evidence and platform-specific verification;
- AI/ML model artifacts, using immutable revision capture, safe-format rules, model evidence, and resource-limited checks.

A cross-cutting post-approval re-evaluation process addresses newly discovered supply-chain compromise, updated detections, signature or publisher concerns, and mutable-source-reference drift.

## Documents in this repository

| Document | Purpose |
|---|---|
| [`package-intake-architecture.md`](package-intake-architecture.md) | The control-focused architecture: stages, control objectives, data flows, and the artifact lifecycle state machine. Product-neutral where possible. |
| [`solution-architecture-tooling.md`](solution-architecture-tooling.md) | The implementation guide: specific tools, licensing/edition trade-offs, deployment commands, and phased rollout plan. |
| [`architecture-tooling-review.md`](architecture-tooling-review.md) | An independent review of the two documents above, identifying factual, licensing, and implementability corrections. Track which findings have been addressed via each document's revision-history table. |
| [`adr/`](adr/README.md) | Architecture Decision Records — one file per undecided or settled cross-cutting decision (status, context, considered options with pros/cons, outcome, consequences). This is where open questions live now; neither main document carries an inline "open issues" section anymore. |

Read the architecture document first for the control model, then the tooling guide for how to actually build it. The review document is worth reading alongside both — several of its findings are still open decisions rather than settled corrections, and are now tracked as ADRs rather than inline text in either document.
| [`package-intake-architecture.md`](package-intake-architecture.md) | Control-focused target architecture: stages, objectives, data flows, artifact paths, and open decisions. |
| [`solution-architecture-tooling.md`](solution-architecture-tooling.md) | Tooling and implementation guide: product choices, licensing considerations, integration guidance, and rollout phases. |
| [`architecture-tooling-review.md`](architecture-tooling-review.md) | Independent review of factual, licensing, implementation, and maintainability risks. |
| [`poc-deployment-plan.md`](poc-deployment-plan.md) | Recommended proof-of-concept scope, identity model, container and VM topologies, sizing, use cases, failure simulations, acceptance criteria, and alternatives. |
| [`poc-build-runbook.md`](poc-build-runbook.md) | POC build procedure, service layout, safe test fixtures, operating runbook, detailed demo script, troubleshooting, reset, and shutdown actions. |

## Recommended reading order

1. Read `package-intake-architecture.md` for the target control model.
2. Read `architecture-tooling-review.md` for the important caveats and decisions that remain open.
3. Read `solution-architecture-tooling.md` for production-oriented implementation options.
4. Read `poc-deployment-plan.md` for the deliberately reduced demonstration architecture.
5. Use `poc-build-runbook.md` to implement and run the POC.

## POC recommendation in one paragraph

The first demonstration should use a containerized, offline-first stack with Keycloak, a small request and approval portal, PostgreSQL, an S3-compatible object store, a hardened fetch worker, ClamAV, YARA, and optional Syft/Grype. Demonstration artifacts should be served from a local mock source so the happy path, integrity failures, EICAR detection, unsafe model rejection, SSRF blocking, scanner outage, and synthetic post-approval recall are deterministic and safe. Dependency-Track, enterprise directory federation, Nexus, observability, and dynamic-analysis VMs can be added as optional profiles after the core request-to-recall loop is stable.
