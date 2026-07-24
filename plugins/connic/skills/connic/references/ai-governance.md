# AI Governance

Canonical docs: `https://connic.co/docs/v1/platform/ai-governance`

AI Governance is an Enterprise documentation and evidence workspace for EU AI Act readiness. It creates versioned readiness records and evidence; it does not make legal determinations or certify compliance.

## Contents

- [Access and permissions](#access-and-permissions)
- [AI systems](#ai-systems)
- [Preliminary assessments](#preliminary-assessments)
- [Controls](#controls)
- [Article 50 records](#article-50-records)
- [Monitoring and incidents](#monitoring-and-incidents)
- [Evidence snapshots](#evidence-snapshots)
- [REST API and workflow](#rest-api-and-workflow)
- [Legal boundaries](#legal-boundaries)

## Access and permissions

AI Governance requires an Enterprise plan. Four project permissions control access:

| Permission | Allows |
| --- | --- |
| `compliance.view` | Read systems, assessments, controls, transparency records, monitoring plans, incidents, and snapshots |
| `compliance.manage` | Manage systems, assessments, controls, transparency policies and attestations, and monitoring plans |
| `compliance.export` | Capture and download evidence snapshots |
| `compliance.incidents.manage` | Create and update incident records and notification evidence |

Every mutation is written to the project audit log with the actor, action, and before/after state.

## AI systems

An AI system is a project-scoped use-case record, independent of any single agent. Record its purpose, owner, geographies, and affected persons, then link the environments, deployments, and agents that implement it. Those links scope telemetry, disclosure evidence, and incident references.

| State | Meaning |
| --- | --- |
| `draft` | Being described; only a pristine draft with no governance records can be deleted |
| `active` | Currently in use |
| `retired` | No longer in use; preserves governance history |

## Preliminary assessments

Assessments are immutable, versioned answer sets covering EU scope, operator roles, prohibited practices, high-risk routes, and Article 50 dimensions. Connic derives a deterministic preliminary result using a pinned catalog version and records the reviewed regulatory source.

- New answers create a new version; assessments are never edited in place.
- Only the latest version can be approved.
- Approval requires a rationale, at least one supported operator role, and acknowledgment of uncertain answers.
- Provider and deployer roles are supported end to end.
- Authorised representative, distributor, importer, and product manufacturer roles are recorded but block approval, control generation or refresh, and evidence export until resolved.

## Controls

Generate controls from the latest approved assessment. Each control records its governing article, applicability rationale, accountability, status, and evidence state.

| Implementation type | Accountability |
| --- | --- |
| `connic` | Connic provides the mechanism |
| `customer` | The organization owns the obligation |
| `shared` | Connic supports an obligation the organization still owns |
| `external` | The obligation is handled outside Connic |

Control status is `gap`, `needs_evidence`, `configured`, or `not_applicable`. Marking a control `not_applicable` requires an exception rationale. Refreshing controls re-derives them from the latest approved assessment while preserving change history.

## Article 50 records

Each system can have one transparency policy covering direct interaction, synthetic-content marking, deepfakes, biometric and emotion use, and public-interest text. Record disclosure copy, locales, placement, frequency, display mode, implementation, and verification.

Connic stores policy and evidence. It does not alter prompts, outputs, streams, or connector traffic; inject notices or metadata; or watermark media. The application remains responsible for implementing and delivering disclosures and markings.

Transparency applications record whether a disclosure was `applied`, `failed`, or `delivery_unknown`. Use `customer_attestation` or `upstream_attestation` provenance and record the attesting party, timestamp, evidence reference, and factual basis. Reusing an idempotency key with identical content returns the existing record; conflicting reuse is rejected.

## Monitoring and incidents

Each system can have a monitoring plan with an owner, review cadence, signals, thresholds, and next-review date. An active plan needs an owner, a next-review date, and either signals or a documented monitoring procedure.

Incidents record awareness time, severity, causal-link and reportability status, corrective actions, notification history, and optional run, environment, or deployment references. Marking an incident `reported` or `closed` requires authority-notification evidence with timestamp, recipient or authority, channel, and external reference.

Connic records the supplied reporting deadline and basis. It does not calculate deadlines or determine notification sufficiency.

## Evidence snapshots

Snapshots are immutable, metadata-only captures of selected systems and environments over an optional time window.

- They are content-addressed using SHA-256 over canonical serialization.
- They cannot be edited or deleted and persist for the project lifetime.
- They exclude raw prompts and model outputs.
- They include systems, latest assessments, controls, transparency records, monitoring plans, incidents, and scoped run, approval, and judge telemetry.
- Environment-scoped exports mark project-only audit telemetry unavailable rather than substituting a project-wide count.

Default limits when a plan does not override them:

- 100 snapshots per project
- 10 MiB per snapshot
- 100 MiB total snapshot storage per project

Downloads are ZIP archives with a human-readable report, canonical JSON, collection CSV/JSON files, and a manifest of per-file SHA-256 hashes. The checksums detect changes; they are not signatures or certifications.

## REST API and workflow

The project-scoped API prefix is `/v1/projects/{project_id}/compliance`. Main route families:

- `/overview`
- `/systems` and `/systems/{id}`
- `/systems/{id}/assessments` and `.../review`
- `/controls`, `/systems/{id}/controls/generate`, and `.../refresh`
- `/systems/{id}/transparency-policy` and `.../transparency-applications`
- `/systems/{id}/monitoring-plan`
- `/incidents` and `/incidents/{id}`
- `/evidence-snapshots` and `/evidence-snapshots/{id}/download`

Typical workflow:

1. Register the system and link its agents, deployments, and environments.
2. Record a preliminary assessment.
3. Approve the latest version with rationale and uncertainty acknowledgment.
4. Generate controls and attach evidence.
5. Document transparency policy and attestations.
6. Maintain monitoring and incident records.
7. Capture and export an evidence snapshot.

## Legal boundaries

AI Governance provides readiness records and evidence, not legal advice, legal determinations, certifications, or professional counsel. Final classification, reporting deadlines, authority-notification sufficiency, and implementation of disclosures remain the organization's responsibility. Use Regulation (EU) 2024/1689 as the primary legal source.
