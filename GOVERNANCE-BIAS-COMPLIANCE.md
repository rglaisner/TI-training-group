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
  - `riskTags` (`string[]`): scenario-level risk markers used by governance and analytics.
  - `lastReviewedAt` (`Timestamp`): governance review time.
  - `lastReviewedBy` (`string`): UID of last reviewer.
  - `lastReviewSummary` (`string`): short free-text governance note (e.g. “Checked for demographic stereotypes; aligned with diversity policy v2.1”).
  - `version` (`number`): governance document version (not scenario content version).

Writers MUST always set `scenarioId` and `governanceStatus` for new governance documents. Readers MUST treat all fields as optional and handle missing values with safe defaults (for example, treating a missing `governanceStatus` as `"draft"` for newly created scenarios) to avoid breaking existing flows.

**Recommended MVP `riskTags` taxonomy** (non-exhaustive, additive):

- `bias_sensitive`: scenario touches demographics, DEI, or fairness topics and requires extra bias review.
- `pii_adjacent`: scenario uses realistic company, market, or people data that is close to PII (but still anonymised).
- `high_impact_label`: scenario can materially change a user’s label tier in a single run.
- `regulatory_thematic`: scenario is explicitly about regulation, compliance, or legal risk.
- `external_data_dependency`: scenario heavily depends on external feeds or integrations whose drift could change outcomes.

Scenarios MAY use multiple tags. Additional organisation-specific tags MAY be introduced, but should be rare and preferably prefixed (e.g. `custom_acme_policy_x`) so they are easy to filter in analytics.

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

#### 2.4 Review SLAs (MVP)

To keep governance lightweight but predictable in V1, scenarios SHOULD follow these minimum review cadences:

- **Baseline re-review**:
  - All `approved` scenarios SHOULD be re-reviewed at least once every **12 months**.
- **High-risk content**:
  - Scenarios with `riskTags` including `bias_sensitive`, `pii_adjacent`, or `high_impact_label` SHOULD be re-reviewed at least once every **6 months**.
- **Trigger-based reviews**:
  - Any material change to scenario content, competency mappings, or label impact SHOULD move the scenario back to `in_review` and trigger a fresh governance review, regardless of timing.

Implementations MAY encode these SLAs as reminders or dashboards; they are process expectations rather than hard runtime enforcement. When `lastReviewedAt` is missing, governance tools SHOULD treat the scenario as “due for initial review”.

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

**Recommended MVP `reasonCode` taxonomy** (usable across admin actions, not just labels):

- Label / profile specific:
  - `external_evidence`: off-platform evidence (e.g. real-world performance) supports the change.
  - `bug_fix`: correcting a known defect in scenario, competency, or label logic.
  - `data_correction`: fixing incorrect or duplicated simulation data (e.g. xp, completions).
  - `appeal_upheld`: user appeal was accepted after review.
  - `appeal_rejected`: user appeal was reviewed and rejected.
  - `abuse_misconduct`: label change due to confirmed abuse, misconduct, or policy violation.
- Platform / tenant operations:
  - `admin_error`: undoing a mistaken admin action (e.g. wrong scenario enabled).
  - `tenant_request`: change initiated by a tenant admin request (e.g. export, deletion).
  - `legal_request`: change driven by legal, regulatory, or data-subject request.
  - `data_minimisation`: proactive reduction of stored data for privacy reasons.
  - `operational_migration`: data or config moved as part of an infrastructure migration.

Implementations MAY extend this list over time, but SHOULD prefer these canonical values so analytics and audit tools can group and filter consistently. Readers MUST treat unknown `reasonCode` values as valid strings and fall back to `freeTextReason` for interpretation.

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

#### 5.3 Bias Review Checklist Fields (MVP)

To make bias reviews more consistent and queryable, reviewers MAY capture findings in a small structured checklist stored inside the scenario governance document.

- **Data shape (conceptual)**:
  - `biasChecklist` (optional) in `governance/{appId}/scenarios/{scenarioId}`:
    - `version` (`string`): checklist version identifier (e.g. `"v1"`).
    - `items` (`{ id: BiasChecklistItemId; status: "pass" | "fail" | "n/a"; notes?: string }[]`).
- **Recommended MVP `BiasChecklistItemId` values**:
  - `representation_reviewed`: scenario representation (characters, organisations, regions) was checked for balance.
  - `stereotypes_checked`: explicit or implicit stereotypes were checked and removed or contextualised.
  - `language_tone_checked`: language tone (including jokes and metaphors) was checked for respect and inclusion.
  - `proxy_variables_checked`: use of proxies for protected characteristics (e.g. postcode, school) was evaluated.
  - `outcome_parity_considered`: likely differential outcomes across cohorts were considered and documented.

All fields in `biasChecklist` are optional. Readers MUST assume that a missing checklist means “no structured checklist recorded yet” and SHOULD fall back to `lastReviewSummary` for human context.

#### 5.4 Logging for Differential Outcomes (No PII)

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

#### 5.5 Automated Gemini-Based Bias Review (MVP)

For V1, any automated Gemini-based bias checks MUST be:

- **Offline and sampled**:
  - Run as background jobs, not per-player, per-event calls.
  - Operate on a capped sample of scenarios and nodes (e.g. up to N high-risk scenarios and K nodes per run).
  - Prioritise scenarios with `riskTags` such as `bias_sensitive`, `pii_adjacent`, or `high_impact_label`.
- **Governed and auditable**:
  - Each automated run SHOULD emit an `admin_action` event with:
    - `adminActionType = "bias_review_run"`.
    - `scenarioId` (if focused on a single scenario) or a list of scenario ids in the payload.
    - `sampledNodeCount`, `modeUsed` (e.g. Gemini model/version), and a coarse `scanOutcome` (`"no_major_issues" | "issues_flagged" | "needs_full_review"`).
    - `reasonCode` (e.g. `data_minimisation`, `tenant_request`, or `policy_update`) where applicable.
  - Summary findings SHOULD also update optional fields on the scenario governance document, for example:
    - `lastBiasScanAt` (`Timestamp` or ISO string).
    - `lastBiasScanSummary` (`string`).
    - `lastBiasScanToolVersion` (`string`).
- **Free-tier friendly**:
  - Platforms SHOULD bound the frequency (e.g. monthly or quarterly runs) and volume (maximum prompts per run) to stay within Gemini free-tier limits.
  - Automated findings are advisory; human reviewers remain responsible for final `governanceStatus` changes and bias decisions.

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

#### 6.4 Tenant-Level Retention & Export Configuration (Data Shape)

To let future Admin (D) and backend agents implement tenant-specific policies, retention and export preferences SHOULD be represented explicitly in governance documents.

- **Collection**: `governance/{appId}/tenants/{tenantId}`
- **Shape (v1, all fields optional)**:
  - `rawEventTtlDays` (`number`): how long raw events SHOULD be retained before archival or deletion (e.g. `365`).
  - `archiveEventsAfterDays` (`number`): when raw events SHOULD be moved to cheaper storage (e.g. `365`) if archiving is supported.
  - `profileTtlDays` (`number`): how long inactive profiles SHOULD be kept (e.g. `null` or omitted means “until tenant deletion”).
  - `governanceTtlDays` (`number`): optional retention window for governance metadata; if omitted, platform defaults (e.g. 7 years) apply.
  - `allowSelfServiceExport` (`boolean`): whether tenant admins can trigger exports from the console.
  - `defaultExportFormat` (`"json" | "csv" | "parquet"`): preferred export format when multiple are available.
  - `lastPolicyUpdatedAt` (`Timestamp`): when the policy was last changed.
  - `lastPolicyUpdatedBy` (`string`): UID of the admin who last changed it.

Readers MUST treat missing fields as “use platform defaults” and MUST NOT assume that a tenant has opted into stricter retention or export controls unless these fields are explicitly set. Changes to these documents SHOULD emit `admin_action` events with `adminActionType = "retention_policy_updated"` and an appropriate `reasonCode`.

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

---

### Appendix A – Shared Helpers & Data Shapes (MVP)

This appendix defines shared interfaces that other agents can rely on. These are **conceptual contracts** rather than concrete implementations; they are compatible with the current in-browser/Firebase architecture and a future API layer.

#### A.1 Scenario Governance Types

- **Risk tags**:

  ```ts
  type RiskTag =
    | "bias_sensitive"
    | "pii_adjacent"
    | "high_impact_label"
    | "regulatory_thematic"
    | "external_data_dependency"
    | (string & {});
  ```

- **Bias checklist**:

  ```ts
  type BiasChecklistItemId =
    | "representation_reviewed"
    | "stereotypes_checked"
    | "language_tone_checked"
    | "proxy_variables_checked"
    | "outcome_parity_considered";

  interface BiasChecklistItem {
    id: BiasChecklistItemId;
    status: "pass" | "fail" | "n/a";
    notes?: string;
  }

  interface BiasChecklist {
    version: string;
    items: BiasChecklistItem[];
  }
  ```

- **Scenario governance document (Firestore-backed)**:

  ```ts
  interface ScenarioGovernanceDoc {
    scenarioId: string;
    governanceStatus: "draft" | "in_review" | "approved" | "deprecated";
    releaseChannel?: "experimental" | "beta" | "ga";
    ownerUserId?: string;
    reviewerUserIds?: string[];
    riskTags?: RiskTag[];
    biasChecklist?: BiasChecklist;
    lastReviewedAt?: unknown;        // Firestore Timestamp or ISO string
    lastReviewedBy?: string;
    lastReviewSummary?: string;
    lastBiasScanAt?: unknown;        // Firestore Timestamp or ISO string
    lastBiasScanSummary?: string;
    lastBiasScanToolVersion?: string;
    version?: number;
  }
  ```

Readers MUST treat all optional fields as nullable/missing and use safe defaults (for example, “no risk tags” or “no checklist recorded yet”).

#### A.2 Tenant Retention Policy Type

```ts
interface TenantRetentionPolicy {
  rawEventTtlDays?: number | null;
  archiveEventsAfterDays?: number | null;
  profileTtlDays?: number | null;
  governanceTtlDays?: number | null;
  allowSelfServiceExport?: boolean | null;
  defaultExportFormat?: "json" | "csv" | "parquet";
  lastPolicyUpdatedAt?: unknown;   // Firestore Timestamp or ISO string
  lastPolicyUpdatedBy?: string | null;
}
```

When this document is absent or fields are `null`, platform defaults from section 6.2/6.3 apply.

#### A.3 Helper Function Contracts

These helpers express how governance data should be queried and exposed to tools, analytics, and admin UIs. They do not mandate where the logic runs (client vs backend).

```ts
// Fetch governance metadata for a scenario (or null if none exists yet).
async function getScenarioGovernance(
  appId: string,
  scenarioId: string
): Promise<ScenarioGovernanceDoc | null> {}

// Convenience helper used by tools/analytics to flag high-risk content.
function isHighRiskScenario(
  governance: ScenarioGovernanceDoc | null
): boolean {
  // Recommended implementation:
  // return governance?.riskTags?.some((tag) =>
  //   ["bias_sensitive", "pii_adjacent", "high_impact_label"].includes(tag)
  // ) ?? false;
}

// Retrieve tenant-specific retention/export preferences (or null/defaults).
async function getTenantRetentionPolicy(
  appId: string,
  tenantId: string
): Promise<TenantRetentionPolicy | null> {}
```

Implementations MAY add convenience helpers (e.g. `listScenariosDueForReview`) built on top of these primitives. All call sites MUST be defensive against `null`/missing governance documents and default to safe, conservative behaviour.

