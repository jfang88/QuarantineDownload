# ADR-0009: Alert-tuning ownership and cadence

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-08-02 |
| Deciders | Security Architecture |
| Related | `solution-architecture-tooling.md` § Phase 8 Hardening, § `fp_suppressions` table |

## Context

The tooling guide's hardening phase calls for tuning YARA rules and Grype thresholds "to reduce noise," but assigns no ownership or review cadence — which is exactly the condition under which alert fatigue quietly erodes a control's effectiveness over time (analysts habitually dismiss a noisy signal, and by the time it matters, a real finding gets the same treatment as the noise). The `fp_suppressions` table introduced during the review response gives the underlying data to track this as a metric; it does not by itself assign a person or a cadence to look at it.

## Decision drivers

- Alert fatigue is a slow-onset failure mode — it needs a standing responsibility, not a one-time fix, to stay controlled.
- Should reuse an existing review cadence (e.g., the CMDB quarterly review) where reasonable, rather than inventing a new recurring meeting.

## Considered options

### Option A: Assign alert-tuning as a standing responsibility of a named role (e.g., the security architecture team lead), with a monthly review of Stage 11b false-positive rates

**Pros**
- Clear, single accountable owner — no ambiguity about whose job this is.
- Monthly cadence is tight enough to catch drift before it compounds across a quarter.

**Cons**
- Adds a new recurring commitment distinct from other existing review cadences (the CMDB review is quarterly) — more meetings/process surface than Option B.

### Option B: Track false-positive suppression counts (recorded per the recheck datastore disposition field) as a formal metric with a target ceiling, reviewed at the same cadence as the CMDB quarterly review

**Pros**
- Reuses an existing review cadence rather than creating a new one.
- A target ceiling gives an objective trigger for action, not just a subjective "does this feel noisy" judgment call.

**Cons**
- Quarterly cadence is looser than monthly — alert fatigue could build for up to a quarter before the formal review catches it, though the target-ceiling metric partially mitigates this by making drift measurable between reviews even if not formally reviewed until the quarterly meeting.

## Decision outcome

*Undecided — pending review meeting sign-off.* Options A and B are not mutually exclusive: a named owner (A) using the target-ceiling metric (B) at whatever cadence is decided is a plausible combined outcome, and probably the intended one, but the specific owner, the specific ceiling value, and the specific cadence (monthly vs. quarterly vs. some other interval) all still need to be set.

## Consequences

*To be completed once a decision is recorded.*
