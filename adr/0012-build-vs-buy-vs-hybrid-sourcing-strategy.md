# ADR-0012: Build-versus-buy-versus-hybrid sourcing strategy for enterprise artifact ingress

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-08-02 |
| Deciders | Security Architecture, Procurement, Engineering leadership |
| Related | [`enterprise-build-vs-buy-evaluation.md`](../enterprise-build-vs-buy-evaluation.md) (full evaluation — this ADR summarizes it, it does not replace it); `package-intake-architecture.md`; `solution-architecture-tooling.md` § Preference |

## Context

The original package-intake documents — `package-intake-architecture.md` and `solution-architecture-tooling.md` in particular — were written from a "build" starting assumption: self-hosted, mostly-free/open-source components (Nexus, Syft, Grype, ClamAV, YARA, CAPE, GitLab CE), assembled and integrated by the organization. `solution-architecture-tooling.md`'s own stated preference is explicit: *"Free, self-hosted tools are the primary recommendation."*

`enterprise-build-vs-buy-evaluation.md` steps back from that assumption and asks the prior question for **artifact and general-file ingress**: should the organization build this at all, buy mature commercial platforms (JFrog, Sonatype, OPSWAT, ReversingLabs Spectra Assure, ServiceNow, Palo Alto Prisma AIRS) that cover large parts of the same requirement, or use some mix of the two? Its own conclusion is a **hybrid** direction — buy repository, binary-analysis, file-ingress, workflow, and model-security capabilities where commercial products are mature, and build only a thin enterprise-owned control plane for canonical identity, lifecycle state, and recall.

This remains the largest sourcing decision for the **ingress domain**: it determines whether `solution-architecture-tooling.md`'s free-first tool list is the actual implementation plan, a partial implementation plan for the "build" portion of a hybrid architecture, or largely superseded by commercial procurement.

### Scope boundary introduced by the controlled-data-release architecture

The repository now also contains `controlled-data-release-architecture.md` and `controlled-data-release-tooling.md`. Those documents address operational-data egress/cross-domain release and have materially different product categories and security objectives (DLP, redaction, controlled collection, MFT/transfer brokering, destination controls). **This ADR does not silently decide the sourcing strategy for that sibling control plane.** The controlled-data-release sourcing decision is tracked separately in [ADR-0014](0014-controlled-data-release-sourcing-strategy.md), while the relationship between the two control planes is tracked in [ADR-0013](0013-separate-ingress-and-data-release-control-planes.md).

## Decision drivers

- Engineering capacity: whether the organization can staff continuous scanner, feed, sandbox, and ruleset operations indefinitely, versus procuring supported products.
- Time to initial capability: a hybrid or buy approach reaches a working control for package/file ingress much faster than building every component.
- Data residency, sovereignty, and air-gap requirements, which may rule out cloud-dependent commercial options for some or all artifact classes.
- Total cost of ownership over a multi-year horizon, not just up-front licence cost versus "free."
- Whether the organization already operates a strategic platform (ServiceNow, JFrog, Nexus, an endpoint management suite) that a given approach would leverage or duplicate.
- Coverage completeness: no single commercial product covers every artifact class and lifecycle stage in this ingress architecture (see the evaluation's capability comparison, § 9) — some build or integration work is required regardless of the option chosen.

## Considered options

See `enterprise-build-vs-buy-evaluation.md` § 5-6 and § 10 for the full analysis. Summarized:

### Option A: Build (open-source and free components)

**Pros:** lowest up-front licence cost; full control over source code and data; strongest fit for fully disconnected/sovereign environments; matches the detailed implementation already written in `solution-architecture-tooling.md`.

**Cons:** very high engineering effort; the organization owns rule maintenance, false-positive handling, upgrades, backup/DR, and incident response indefinitely; "free" understates true cost once staffing is counted; slow time to initial capability across every artifact class.

### Option B: Buy (commercial platforms)

**Pros:** fast time to value for product-covered paths; stronger commercial threat intelligence and vendor research than most self-hosted equivalents; contractual support and escalation; mature workflow/CMDB/SLA capability if built on an existing ITSM investment.

**Cons:** high licence cost; no single vendor covers the complete lifecycle across all artifact classes (package, general file, business onboarding, model security are distinct commercial markets per the evaluation's § 2 and § 7); deeper product and data-model lock-in; the organization still owns architecture, policy, and cross-product integration regardless of how much is bought.

### Option C: Hybrid (buy mature specialist capabilities, build a thin control plane)

**Pros:** broadest practical coverage; avoids both the "build everything" engineering burden and the "no product covers the whole lifecycle" gap; keeps canonical artifact identity, lifecycle state, and recall logic portable and enterprise-owned even as underlying vendor products change; the evaluation's own capability comparison (§ 9) shows this option scoring "Strong" across the most rows.

**Cons:** inherits both licence and integration cost; requires deliberate scope discipline to avoid duplicating what a bought product already does (the evaluation's § 11 lists this as the main hybrid cost-control risk); still the most complex option to govern, since it spans multiple vendor relationships plus internal engineering.

## Decision outcome

*Undecided — pending review by Security Architecture, Procurement, and Engineering leadership.* `enterprise-build-vs-buy-evaluation.md` § 13 states hybrid as "the working recommendation," which is the evaluation author's analysis, not a ratified organizational decision — this ADR exists specifically so that distinction isn't lost. Follow the evaluation's § 14 "Suggested decision sequence" to reach a decision: confirm artifact classes and volumes in scope, inventory existing strategic platforms, shortlist products per the three **ingress** capability domains, run the mandatory evaluation scenarios (§ 12) against real vendor trials, and model five-year TCO before selecting.

## Consequences

- **If Build is chosen:** `solution-architecture-tooling.md` remains the primary ingress implementation guide largely as written; this ADR should be marked `Accepted` with that outcome, and `enterprise-build-vs-buy-evaluation.md`'s hybrid recommendation noted as considered-and-not-selected with the reason (most likely sovereignty/air-gap constraints or engineering capacity conclusions specific to the organization).
- **If Buy is chosen:** large portions of `solution-architecture-tooling.md` become non-authoritative for whichever ingress capability domains are replaced by commercial product; `package-intake-architecture.md`'s control model should still hold (it is written to be product-neutral), but its "Security controls by stage" sections would need re-mapping to vendor capabilities rather than the specific free tools currently named.
- **If Hybrid is chosen (the current working recommendation):** `solution-architecture-tooling.md` needs a pass to mark which of its current tool recommendations are retained (the thin control-plane pieces: evidence database, lifecycle state machine, GitLab request portal or its replacement) versus superseded by a bought product (most likely Syft/Grype/Dependency-Track if a commercial SCA/repository platform is selected, and CAPE/dynamic-analysis tooling if a commercial file-security platform is selected). This re-mapping is nontrivial follow-up work regardless of which specific vendors are chosen, and should be scoped as its own effort once this ADR is `Accepted`.
- Regardless of outcome, per the evaluation's § 15 references and its own recommendation, further downstream ADRs should be created for the specific repository platform, generic file-ingress product (if bought), binary-analysis product (if bought), and model-security product (if bought) — this ADR records the **ingress sourcing strategy** decision; it does not select specific vendors and it does not decide the sibling controlled-data-release sourcing strategy.
