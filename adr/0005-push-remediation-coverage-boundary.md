# ADR-0005: Push-remediation coverage boundary for unmanaged endpoints

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-08-02 |
| Deciders | Security Architecture, Endpoint Management |
| Related | `package-intake-architecture.md` § Stage 10 Consumption and inventory; `solution-architecture-tooling.md` § Push remediation tooling on recall |

## Context

Stage 10/11b recall workflows are designed to trigger push remediation (via WSUS/SCCM/Intune/Ansible) to known-deployed systems on confirmed compromise, not just send a notification. This works cleanly for any system already enrolled in one of those management tools. It does not work for systems outside that reach: unmanaged developer workstations, contractor laptops, or air-gapped/isolated network segments. The fraction of the target fleet actually enrolled in a management tool has not been confirmed, and that fraction directly determines how big this gap is in practice.

## Decision drivers

- A recall control that silently degrades to "notification only" for some unknown fraction of the fleet is a gap that should be sized and either closed or explicitly accepted — not left as an unstated assumption.
- Endpoint management tool coverage is itself a fact to establish, not a policy choice, and should be measured before choosing among the options below.

## Considered options

### Option A: Scope automated push remediation to managed fleets only; unmanaged endpoints get notification plus a manual-follow-up SLA and an escalation path if missed

**Pros**
- Achievable with current tooling — no new enrollment requirement.
- Explicit SLA and escalation makes the residual gap visible and tracked rather than silent.

**Cons**
- Recall speed and reliability for unmanaged systems remains meaningfully weaker than for managed ones; a determined or careless owner can miss the SLA with no automated backstop.

### Option B: Require enrollment in a management tool as a precondition for approval to consume from the repository at all

**Pros**
- Closes the gap by policy — if you can't be reached for remediation, you can't consume from the approved repository in the first place.
- Simpler operational model going forward (100% push-remediation coverage by construction).

**Cons**
- May not be achievable for legitimate categories of system (some air-gapped or isolated segments by design, certain contractor arrangements) without carving out exceptions, which reintroduces the gap it's meant to close.
- Enrollment as a precondition is an organizational policy change with change-management cost beyond this architecture's scope.

### Option C: Accept the gap for a defined class of exception (e.g., air-gapped environments) with compensating controls (manual periodic audit) instead of push remediation

**Pros**
- Realistic for genuinely unreachable environments (air-gapped by design) where push remediation is not a meaningful concept regardless of policy.
- Compensating manual audit is better than silent gap.

**Cons**
- Requires defining "defined class of exception" precisely enough that it doesn't become a catch-all that erodes Option A/B's coverage guarantees.

## Decision outcome

*Undecided — pending review meeting sign-off, and pending measurement of current management-tool enrollment fraction across the target fleet.* Option A and Option C are not mutually exclusive — A as the general policy, C as a scoped exception for genuinely air-gapped segments, is a plausible combined outcome, but that combination itself needs sign-off rather than being assumed.

## Consequences

*To be completed once a decision is recorded.* At minimum: the enrollment-fraction measurement should happen regardless of which option is chosen, since it's a prerequisite fact for evaluating all three.
