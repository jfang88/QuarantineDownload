# Enterprise Package Intake and Approved Repository — Reference Architecture

> **Status: draft reference architecture and POC design, not a deployed control environment.** This repository documents a proposed design for controlled software acquisition, recommended tooling, commercial build-versus-buy evaluation, review findings, a decision log, and a proof-of-concept plan. Nothing in this repository should be treated as a production control until it has been implemented, tested, approved, and operated under the organisation's governance processes. **9 architecture decisions are still open**, including the top-level build-versus-buy-versus-hybrid sourcing strategy — see [`adr/README.md`](adr/README.md).

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
| [`package-intake-architecture.md`](package-intake-architecture.md) | The control-focused target architecture: stages, control objectives, data flows, artifact paths, and the artifact lifecycle state machine. Product-neutral where possible. |
| [`solution-architecture-tooling.md`](solution-architecture-tooling.md) | The implementation guide: specific tools, licensing/edition trade-offs, deployment commands, and phased rollout plan. |
| [`architecture-tooling-review.md`](architecture-tooling-review.md) | An independent review of the two documents above, tracked as a living document — its own "Resolution status summary" and "Outstanding — needs further work" sections state which of the 18 original findings are fully resolved, partially resolved (with the specific missing piece named), or still open, so a reader doesn't have to cross-reference revision-history tables to find out. |
| [`enterprise-build-vs-buy-evaluation.md`](enterprise-build-vs-buy-evaluation.md) | The enterprise requirements, risks, open-source/free versus commercial comparison, vendor capability landscape, solution patterns, procurement tests, TCO considerations, and recommended hybrid build-versus-buy direction. |
| [`adr/README.md`](adr/README.md) | **The decision register.** Architecture Decision Records — one file per undecided or settled cross-cutting decision (status, context, considered options with pros/cons, outcome, consequences). This is the single place open questions live; neither main document repeats this table inline. |
| [`poc-deployment-plan.md`](poc-deployment-plan.md) | The deliberately reduced proof-of-concept scope: identity model, container/VM topologies, sizing, use cases, failure simulations, acceptance criteria, and alternatives considered. |
| [`poc-build-runbook.md`](poc-build-runbook.md) | The POC build procedure, service layout, safe test fixtures, operating runbook, demo script, troubleshooting, reset, and shutdown actions. |

## Recommended reading order

1. Read `package-intake-architecture.md` for the target control model.
2. Check `adr/README.md` for which parts of that model are still open questions rather than settled — several sections above reflect a working assumption, not a final decision.
3. Read `architecture-tooling-review.md` for the independent review that drove several of those open decisions and the corrections already folded into the two main documents.
4. Read `enterprise-build-vs-buy-evaluation.md` to understand the enterprise requirements and compare open-source, commercial, and hybrid implementation approaches.
5. Read `solution-architecture-tooling.md` for detailed implementation options after the strategic sourcing direction is understood.
6. Read `poc-deployment-plan.md` for the deliberately reduced demonstration architecture, then use `poc-build-runbook.md` to implement and run it.

## POC recommendation in one paragraph

The first demonstration should use a containerized, offline-first stack with Keycloak, a small request and approval portal, PostgreSQL, an S3-compatible object store, a hardened fetch worker, ClamAV, YARA, and optional Syft/Grype. Demonstration artifacts should be served from a local mock source so the happy path, integrity failures, EICAR detection, unsafe model rejection, SSRF blocking, scanner outage, and synthetic post-approval recall are deterministic and safe. Dependency-Track, enterprise directory federation, Nexus, observability, and dynamic-analysis VMs can be added as optional profiles after the core request-to-recall loop is stable. The POC is a limited proof of concept — it validates the control model and evidence flow, not a production build, and does not carry a full test suite for every capability described in the two main documents.
