# ADR-0004: VirusTotal Public API usage and paid-tier procurement trigger

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-08-02 |
| Deciders | Security Architecture, Procurement/Legal |
| Related | `package-intake-architecture.md` § VirusTotal hash lookup, § Stage 11b; `solution-architecture-tooling.md` § VirusTotal hash API |

## Context

The architecture uses VirusTotal's Public API (free, 500 lookups/day) as the default for both Stage 6 intake screening and Stage 11b retroactive recheck. Two separate pressures make this untenable as a permanent default, not just an initial-scale convenience:

1. **Growth, not just a fixed ceiling.** Stage 11b rechecks the *entire* approved inventory on a recurring schedule. Every artifact this pipeline ever promotes becomes a permanent, compounding line item against the daily budget — there is no steady state where "the free tier is enough," only a longer or shorter runway before Tier 2/3 recheck staleness becomes unacceptable.
2. **Terms of service.** VirusTotal's Public API terms restrict use in commercial products, services, or automated business workflows. An unattended, nightly, production-integrated enterprise recheck job is plausibly exactly that, independent of whether the 500/day rate limit is ever actually exceeded.

As of `package-intake-architecture.md` v1.10, Stage 11b applies an age-based tapering recheck cadence (daily for the first 30 days after promotion, weekly through 180 days, monthly after) as a partial mitigation. This caps the steady-state size of the "always daily" cohort rather than letting it track total inventory size, so it slows the rate-limit pressure described above — but it does not resolve either pressure on its own: the 31+ day cohorts still grow with inventory, and cadence has no bearing on the ToS question. Treat the tapering design as a mitigation that changes the timeline of the options below, not as an answer to this ADR.

## Decision drivers

- Avoid discovering the rate-limit ceiling in production as an outage rather than a planned procurement event.
- Resolve the ToS compliance question on its own timeline, not only when the rate limit forces the issue.
- Minimize unnecessary fixed cost if usage stays genuinely small for a long time.

## Considered options

### Option A: Set a fixed inventory-size threshold that triggers automatic budget approval for the paid tier

**Pros**
- Predictable, plannable — procurement isn't a surprise when it fires.
- Simple to implement as a monitoring alert on inventory count.

**Cons**
- Doesn't resolve the ToS question independently — an organization could stay under the size threshold indefinitely while still operating outside the Public API's permitted use.
- Threshold picked today may be wrong for actual future growth rate; needs periodic revisiting either way.

### Option B: Accept degraded Tier 2/3 recheck frequency indefinitely; treat VirusTotal as Tier-1-only and rely on MalwareBazaar (no meaningful rate limit) plus YARA re-scan as the primary Stage 11b signal for lower tiers

**Pros**
- No new spend.
- Reduces how fast the rate-limit problem grows, since only the highest-risk tier competes for the VT budget.

**Cons**
- Does not resolve the ToS question at all — the Public API is still being used in an automated business workflow, just at lower volume.
- Permanently degrades recheck coverage for Tier 2/3 artifacts, which is a security posture decision being made as a side effect of a cost-avoidance choice rather than deliberately.

### Option C: Replace VirusTotal with a commercial threat-intel feed (ReversingLabs TitaniumCloud, Recorded Future, or VirusTotal Premium) sized for bulk queries from the start

**Pros**
- Resolves both the rate-limit and ToS problems at once.
- Commercial feeds typically offer richer signal (reputation scoring, campaign attribution) beyond raw AV engine counts.

**Cons**
- Highest fixed cost of the options, paid regardless of whether current volume would have stayed under the free tier's limits for years.
- Vendor selection and integration work not yet scoped.

### Option D: Get a legal/procurement read on current Public API usage now, independent of the rate-limit question

**Pros**
- Directly addresses the ToS risk, which is a compliance question that doesn't go away regardless of which of A/B/C is chosen for the rate-limit question.
- Cheap — a legal review, not an engineering change.

**Cons**
- Doesn't answer the capacity question on its own; needs to be paired with A, B, or C.

## Decision outcome

*Undecided — pending review meeting sign-off.* Option D should happen regardless of which capacity option is chosen, since it addresses a distinct question (compliance, not capacity). Among A/B/C, the choice depends on the organization's actual planned inventory growth rate and risk tolerance for degraded Tier 2/3 coverage, which have not yet been sized against this architecture.

## Consequences

*To be completed once a decision is recorded.* At minimum: whichever option is chosen, the Stage 11b rate-limit budgeting logic and its "Open issues" framing in the architecture/tooling documents should be updated to reference this ADR instead of restating the options.

## Sources

The ToS restriction and 500/day rate limit this ADR is built on were independently verified against [VirusTotal Docs — Public vs Premium API](https://docs.virustotal.com/reference/public-vs-premium-api) — see `package-intake-architecture.md`'s References section for the verification record.
