## Workstream I – Governance, Bias & Compliance Guardrails

This document defines the governance model, bias/fairness guardrails, and compliance patterns that other workstreams must reference. It is designed to be **additive** to the existing runtime and schemas from Workstream C (competencies and labels), Workstream D (tenants/admin), and Workstream E (events & analytics).

---

### 1. Principles

- **Explainability first**: Every label, override, and admin action must be explainable from an audit trail.
- **No raw PII**: The simulation runtime MUST NOT store names, emails, or other directly identifying fields in Firestore profiles, events, or Gemini prompts.
- **Central standards**: Competency/label logic from Workstream C and governance rules from this document are the single sources of truth. Other workstreams MUST NOT invent their own scoring or governance rules.
- **Append-only audit**: Events are append-only; governance never edits or deletes historical events, only appends new ones (data retention is handled via aggregation or archive, not mutation).
- **Tenant & cohort awareness**: Events and profiles carry enough **non-PII** metadata (tenant and cohort identifiers) to support fairness analysis.

---

### 2. Scenario Governance Workflow

#### 2.1 Lifecycle States

Every scenario (`SCENARIOS[id]`) has an associated governance state, managed out-of-band from the scenario definition and stored in Firestore.

- **States**:
  - `draft`: content is under active authoring; not visible to players.
  - `in_review`: frozen for governance/SME review; only accessible in admin/test environments.
  - `approved`: cleared for use in production tenants.
  - `deprecated`: visible in historical analytics but no longer assignable to new tenants/cohorts.

- **Allowed transitions**:
  - `draft → in_review → approved → deprecated`.
  - `approved → in_review` is allowed when making significant content changes (e.g. new label mappings or sensitive topics).

**Contract for Admin Console (Workstream D)**:

- Scenario visibility toggles MUST be gated by `governanceStatus`:
  - Only scenarios with `governanceStatus === "approved"` can be enabled for production tenants.
  - `draft` and `in_review` scenarios MAY be enabled only for explicitly marked `test` tenants.

#### 2.2 Scenario Governance Metadata (Firestore)

Governance metadata is stored separately from scenario content to avoid coupling runtime logic to admin workflows.

- **Collection**: `governance/{appId}/scenarios/{scenarioId}`
- **Shape (v1)**:
  - `scenarioId`* (`string`): mirrors `SCENARIOS[id]`.
  - `governanceStatus`* (`"draft" | "in_review" | "approved" | "deprecated"`).
  - `releaseChannel` (`"experimental" | "beta" | "ga"`).
  - `ownerUserId` (`string`): Firebase UID of primary scenario owner (no names/emails).
  - `reviewerUserIds` (`string[]`): Firebase UIDs of reviewers.
  - `riskTags` (`string[]`): e.g. `["bias_sensitive", "pii_adjacent", "high_impact_label"]`.
  - `lastReviewedAt` (`Timestamp`): governance review time.
  - `lastReviewedBy` (`string`): UID of last reviewer.
  - `lastReviewSummary` (`string`): short free-text governance note (e.g. “Checked for demographic stereotypes; aligned with diversity policy v2.1”).
  - `version` (`number`): governance document version (not scenario content version).

Writers MUST always set `scenarioId` and `governanceStatus` for new governance documents. Readers MUST treat all fields as optional and handle missing values with safe defaults (for example, treating a missing `governanceStatus` as `"draft"` for newly created scenarios) to avoid breaking existing flows.

#### 2.3 Scenario Change Logs & Audit

Scenario content changes MUST be tracked via append-only governance events:

- **Event types** (see section 4 for schema details):
  - `admin_action.scenario_created`
  - `admin_action.scenario_updated`
  - `admin_action.scenario_status_changed`
  - `admin_action.scenario_release_channel_changed`

Each governance event MUST include:

- `scenarioId`, `previousStatus`, `nextStatus` (where applicable).
- `previousReleaseChannel`, `nextReleaseChannel` (where applicable).
- `changeSummary`: short explanation written by the editor/reviewer.
- `changedByUserId`: UID of the admin making the change.

This ensures that a scenario’s lifecycle and governance decisions can be reconstructed from events plus the latest governance document.

---

### 3. Label Governance & Auditability

Workstream C defines how labels of excellence are **computed** from XP and competency evidence. Governance adds rules for how label changes are **recorded** and when manual overrides are allowed.

#### 3.1 Automatic Label Changes

The existing gameplay events already include:

- `labelsBefore` / `labelsAfter`
- `xpBefore` / `xpAfter`
- `competencyDeltas`

**Governance requirement**:

- Every state-changing mission event that can affect labels (`mission_complete`, and any future “recompute” events) MUST populate `labelsBefore` and `labelsAfter`.
- Dashboards and audit tools MUST treat `labelsBefore/After` as the authoritative trail of how labels evolved over time.

#### 3.2 Manual Label Overrides

Admins may occasionally need to adjust labels (e.g. to reflect off-platform evidence or to correct a bug).

- **Admin actions**:
  - `admin_action.label_awarded_manual`
  - `admin_action.label_revoked_manual`
  - `admin_action.label_adjusted_manual` (e.g. tier change).

Each manual label event MUST include:

- `targetUserId` (`string`): UID whose label is affected.
- `labelsBefore` / `labelsAfter` (`string[]`).
- `reasonCode` (`string`): e.g. `external_evidence`, `bug_fix`, `appeal_upheld`.
- `freeTextReason` (`string`): short human-readable justification.
- `changedByUserId` (`string`): UID of the admin.

Manual overrides MUST NEVER mutate historical events. They append new entries to the event stream for downstream analytics and audits.

---

### 4. Governance Event Extensions (Workstreams C, D, E)

The base event schema in `ARCHITECTURE.md` remains the source of truth. Workstream I only adds **optional** fields and new event subtypes.

#### 4.1 Additional Top-Level Fields

The following fields MAY be added to event documents under `events/{appId}/users/{uid}/{eventId}`:

- `tenantId` (`string`):
  - Copied from `adminConfig.activeTenantId` at event time.
  - Used to separate organizations in analytics; not considered PII.
- `cohortIds` (`string[]`):
  - Opaque cohort or archetype identifiers (e.g. `"pilot_2026_batch_a"`, `"ti_collective_cohort_3"`).
  - Defined and managed by admins or integration systems; MUST NOT embed raw PII (no emails, employee ids, etc.).
- `governanceContext` (`{ [key: string]: string | number | boolean }`):
  - Optional bag of small governance flags (e.g. `{ highRisk: true, scenarioRiskTag: "bias_sensitive" }`).

Readers MUST treat all of these fields as optional and default to “unknown” when missing so historical events remain valid.

#### 4.2 Admin / Governance Event Subtypes

Existing `type = "admin_action"` events are extended with a more specific subtype:

- Shared fields:
  - `type` = `"admin_action"`.
  - `appId`, `userId` (admin UID), `createdAt`, `version`.
  - `payload.adminActionType` (`string`).

- **Supported `payload.adminActionType` values**:
  - Scenario lifecycle:
    - `"scenario_created"`, `"scenario_updated"`, `"scenario_status_changed"`, `"scenario_release_channel_changed"`.
  - Label governance:
    - `"label_awarded_manual"`, `"label_revoked_manual"`, `"label_adjusted_manual"`.
  - Governance configuration:
    - `"governance_policy_updated"`, `"persona_guardrail_updated"`, `"retention_policy_updated"`.

Each admin event’s `payload` SHOULD include enough fields to reconstruct:

- What changed (previous vs next values).
- Who changed it (`changedByUserId`).
- Why it changed (`reasonCode`, `freeTextReason`).

---

### 5. Bias & Fairness Guardrails

#### 5.1 Persona Prompt Guardrails

Gemini persona prompts MUST include a central, shared guardrail preamble, in addition to persona-specific tone and context.

- **Guardrail preamble (conceptual)**:
  - Instruct Gemini to:
    - Avoid discriminatory or stereotyped assumptions about protected characteristics (e.g. race, gender, age, disability, religion).
    - Avoid asking for or inferring PII about the player or their organization.
    - Focus feedback on decision quality, reasoning, and business impact, not personal traits.
    - Flag potentially biased or ethically questionable scenario content in a neutral way (for author review, not shown directly as accusatory to players).

**Implementation contract**:

- Workstream B (personas) and any Gemini integration MUST:
  - Inject the shared guardrail preamble into every Gemini call.
  - Treat persona-specific instructions as **additional**, not overriding, constraints.

#### 5.2 Scenario Content Review for Bias

As part of the `in_review → approved` transition:

- Reviewers MUST explicitly check:
  - Whether scenario text or choices rely on stereotypes or proxy variables for protected characteristics.
  - Whether scoring or competency mappings could produce systematically different outcomes for different cohorts with similar performance.
- The `lastReviewSummary` field SHOULD note any bias-related considerations or mitigations.

Future iterations may introduce automated checks (e.g. batch Gemini reviews) but must respect the Gemini free-tier constraint by:

- Sampling scenarios and decisions rather than exhaustively checking every node.
- Running offline review jobs, not per-player, per-event checks.

#### 5.3 Logging for Differential Outcomes (No PII)

To enable fairness analysis across cohorts without storing PII:

- Profiles MAY include:
  - `cohortIds` (`string[]`): same opaque cohort identifiers as in events.
- Event writers SHOULD:
  - Copy `tenantId` and `cohortIds` from the profile into each event.
  - Ensure no names, emails, or direct identifiers are included in `scenarioTags`, `reasonCode`, or other free-text fields.

This allows Workstream E to:

- Compare label attainment and XP progression across tenants and cohorts.
- Detect systematic differences in outcomes (e.g. one tenant’s cohort struggling with a specific scenario) without access to PII.

---

### 6. Privacy & Retention Policies (Initial)

These policies are intentionally minimal but designed to be acceptable for enterprise review and to evolve over time.

#### 6.1 Data Minimization

- Profiles:
  - Store only pseudonymous identifiers (`userId`, optional `tenantId`, optional `cohortIds`) and simulation-related fields (XP, completed scenarios, competency evidence, labels).
  - MUST NOT store names, emails, or free-form personal descriptions.
- Events:
  - MUST NOT include PII in `scenarioTags`, `reasonCode`, or `freeTextReason`.
  - Free-text fields should focus on scenario or governance context, not user identity.
- Gemini:
  - Prompts MUST NOT include PII beyond the player’s role description (e.g. “You are a TI analyst at a mid-sized company”) and should avoid specific organization names unless provided in a synthetic or anonymised way by content authors.

#### 6.2 Retention & Aggregation

- Profiles:
  - Retained for the lifetime of the account or tenant, subject to tenant-level deletion/export requests (to be implemented in Workstream D/Enterprise readiness).
- Events:
  - Raw, per-event logs MAY be retained for a limited window (e.g. 12–24 months), after which:
    - Aggregated metrics (per-scenario, per-tenant, per-cohort) are kept.
    - Raw documents can be archived to cheaper storage or deleted, per tenant policy.
- Governance metadata:
  - Governance documents and governance-related events SHOULD be retained longer (e.g. 7 years) for compliance reviews, but MAY be archived offline if needed.

The precise retention windows and archiving mechanisms WILL be configured per tenant in future enterprise-focused workstreams; for now, this document defines the directional policy for implementers.

#### 6.3 Access & Export

- Admins MAY request:
  - Export of their tenant’s aggregated analytics (scenario usage, labels, competency coverage).
  - Deletion of a tenant’s profiles and events (subject to an offline archival step if required by regulation).
- Any export feature MUST:
  - Respect the no-PII guarantee (exports are keyed by pseudonymous identifiers, tenant ids, and cohort ids).
  - Be logged via `admin_action` events with a clear reason and scope.

---

### 7. Coordination with Other Workstreams

- **Workstream C (Competencies & Labels)**:
  - Uses this governance spec to:
    - Ensure label rule changes are captured as governance events.
    - Provide human-readable explanations that align with audit trails.
- **Workstream D (Admin & Multi-Tenant)**:
  - Owns the Admin Console features for:
    - Editing scenario governance metadata.
    - Managing tenants, cohorts, and data retention options.
- **Workstream E (Events & Analytics UX)**:
  - Uses the extended event schema (tenant and cohort metadata, admin_action subtypes) to:
    - Build fairness and governance dashboards.
    - Provide evidence for calibration of competencies and labels.

All new runtime or admin features that affect scenarios, labels, or governance MUST reference this document and emit appropriate events so that bias, fairness, and compliance can be assessed retrospectively.

