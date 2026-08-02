# ADR-0007: Model registry tooling — Nexus tags + CMDB vs. a dedicated platform

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-08-02 |
| Deciders | Security Architecture, ML Platform team (if one exists) |
| Related | `package-intake-architecture.md` § Path C, § Stage 5c; `solution-architecture-tooling.md` § Stage 5c ML-BOM and model card capture |

## Context

Path C's model metadata (model card, license, base-model lineage, pinned hub revision, ML-BOM) currently piggybacks on Nexus tags plus the CMDB, mirroring the pattern used for proprietary binaries in Path B. A dedicated model registry (MLflow, or a model-specific module in an existing MLOps platform) would give richer lineage graphs — which fine-tune came from which base model, which training experiment produced which checkpoint — and native integration with a training/fine-tuning pipeline, at the cost of standing up and securing a new system of record.

## Decision drivers

- Whether the organization does meaningful in-house fine-tuning (where lineage tracking is a core workflow need, not an add-on) versus primarily consuming pre-trained models as-is.
- Cost of standing up, securing, and backing up a new system of record versus reusing infrastructure this architecture already requires (Nexus, CMDB, the evidence database).
- Migration cost if the Nexus-tags-plus-CMDB approach is chosen first and later needs to move to a dedicated registry once model volume or lineage complexity outgrows it.

## Considered options

### Option A: Start with Nexus tags plus CMDB (no new system); revisit if model volume or lineage complexity outgrows it

**Pros**
- No new system to secure, back up, or operationally own.
- Consistent with the Path B pattern already established for proprietary binaries — one fewer novel concept in the architecture.
- Reuses the per-artifact evidence database (ADR-0003 context) as the natural home for the richer record, same as every other artifact type.

**Cons**
- Lineage relationships (base model → fine-tune → further fine-tune, which experiment produced which checkpoint) are graph-shaped data that a flat evidence-database record represents awkwardly compared to a purpose-built registry.
- If the organization is already committed to meaningful in-house fine-tuning, this option creates migration work later that could have been avoided by starting with the dedicated platform.

### Option B: Stand up a dedicated model registry (e.g., MLflow) from the start

**Pros**
- Native lineage graph support fits the actual shape of fine-tuning/training relationships.
- If in-house fine-tuning is already planned, this avoids a future migration.

**Cons**
- New system of record: needs its own security review, backup/DR plan (per the architecture's systems-of-record backup requirements), and access-control model — real cost even before considering integration work.
- Premature if the organization's actual usage is closer to "consume pre-trained models as-is" than "run an in-house fine-tuning pipeline" — the lineage-graph benefit isn't being used.

## Decision outcome

*Undecided — pending confirmation of whether meaningful in-house fine-tuning is actually planned.* This is the deciding fact, not a judgment call independent of it: Option A is the right default in the absence of confirmed in-house fine-tuning plans; Option B is justified only once those plans are real, not merely possible.

## Consequences

*To be completed once a decision is recorded.* If Option A is chosen initially, define in advance what "model volume or lineage complexity outgrows it" concretely means (e.g., a fine-tune lineage more than N generations deep, or more than N models under active management) so the revisit trigger isn't left as vague as the original open-issue framing.
