## TI Simulation Platform – Runtime Architecture & Domain Model

### Runtime Architecture (Current Prototype)

- **Client-only web app** rendered from `index.html`:
  - UI built with React (UMD build) and TailwindCSS.
  - All game logic runs in the browser.
  - Firebase is used as the persistence and event backend.
- **Backend via Firebase**:
  - **Authentication**: anonymous sign-in using Firebase Auth.
  - **Storage**: Firestore documents for:
    - User profile and progress: `artifacts/{appId}/users/{uid}/profile`.
    - Structured events: `events/{appId}/users/{uid}/{timestamp}`.

### Core Domain Objects (implemented in `index.html`)

- **Scenario / Mission (`SCENARIOS`)**
  - `id`, `title`, `subtitle`, `difficulty`, `accent`, `xpReward`, `brief`.
  - `nodes`: map of node id → `{ scene, choices?, isEnd? }`.
  - Optional `tools.enabledToolIds`: `string[]` — limits which tools appear in the sidebar for this mission; if absent, all default tools are available. Used by `getToolsForScenario`.
  - Behaves as a mission/arc consisting of multiple decision nodes.

- **Node**
  - Represents a single point in the storyline.
  - `scene`: narrative text presented to the user.
  - Optional `externalData.requests`: array of per-node external data requests (e.g. for Market & Comp Visualizer); structure is node-level, consumed by tools; SDL loaders may populate this. No fetcher implementation required for MVP.
  - `choices`: array of structured actions:
    - `text`: what the user selects.
    - `next`: id of the next node.
    - `xp`: XP reward (optional, defaulted in the engine).
    - `tag`: qualitative marker (e.g. `BEST PRACTICE`, `INTEGRITY`).
  - `isEnd`: marks the end of the scenario.

- **Game State (`gameState` / profile document)**
  - `xp`: numeric experience points accumulated across scenarios.
  - `completed`: map of `scenarioId → true`.
  - `meta`: structured simulation metadata (e.g. timeline week, organisational risk, persona trust).
  - `competencyEvidence`: map of `competencyId → { count, lastUpdated, lastScenarioId }` maintained by Workstream C.
  - Persisted in Firestore as the user profile document at `artifacts/{appId}/users/{uid}/profile`.

- **Competency Catalog (`COMPETENCIES`)**
  - Central definition of the TI / Workforce Planning / People Analytics competency model.
  - Each competency has:
    - `id`: stable identifier used in events and profiles (e.g. `data_integrity_ethics`).
    - `label`: short human-readable name.
    - `domain`: broad area (e.g. Talent Intelligence, People Analytics, Workforce Planning).
    - `description`: explanation of what “good” looks like.
    - `tags`: array of scenario decision tags (e.g. `INTEGRITY`, `TEAM_LEADERSHIP`) that count as evidence for this competency.
  - A derived map links tags to competencies:
    - `TAG_TO_COMPETENCY_IDS[tag] → string[]` of competency ids.
  - **Contract for other workstreams**:
    - New scenario tags SHOULD be mapped here rather than ad hoc.
    - Analytics (Workstream E) MUST treat `competencyId` values as opaque ids that come from this catalog.

- **Label of Excellence (`labels`)**
  - Computed from XP, number of completed scenarios, and breadth/depth of `competencyEvidence`:
    - `ti_analyst`: early label for repeated, basic success with at least minimal competency evidence (or XP-only fallback for legacy profiles).
    - `ti_strategist`: mid-tier label, requires higher XP, mission breadth, and evidence across multiple competencies.
    - `ti_architect`: top-tier label, requires high XP, breadth, and repeated evidence across many competencies.
  - Each label includes:
    - `id`, `name`, `tier`.
    - `explanation`: plain-language summary of why the user currently holds this label.
    - `nextSteps`: guidance on what to do next to grow or sustain the label.
  - Stored as an array in Firestore profile and surfaced in the Rank view.

- **Module (`MODULES` & `adminConfig.modules`)**
  - Encapsulates platform capabilities:
    - `core_simulation`, `mentor_persona`, `voice_overlay`, `team_mode`.
  - Each module has:
    - `id`, `name`, `description`, `defaultEnabled`.
  - Admin configuration:
    - `modules[id] = boolean` to enable/disable modules at runtime.
    - Controlled via the Admin Console (no deploy needed to toggle).

- **Persona (`PERSONAS` & `adminConfig.activePersonaId`)**
  - AI persona definitions used to condition Gemini feedback:
    - `id`, `label`, `role`, `tone`, `perspective`.
  - Admin chooses the active persona; the simulation engine injects persona
    context into the prompt so the same scenarios can be experienced through
    different “voices” (mentor, boss, peer).

### Simulation Engine (implemented via helpers)

- **State transitions**
  - `startSim(scenario)`:
    - Resets local interaction state.
    - Optionally plays TTS for the first scene (if `voice_overlay` is enabled).
    - Logs a `mission_start` event for analytics.
  - `handleDecision(choice)`:
    - Optionally calls Gemini (if `mentor_persona` enabled) with persona context.
    - Appends to local `history`.
    - If `next` node is terminal:
      - Computes new XP and completed scenarios.
      - Re-computes labels (`computeLabels`).
      - Persists profile + labels in Firestore.
      - Logs `mission_complete`.
    - If `next` node is non-terminal:
      - Moves to the next node and (optionally) plays TTS.
      - Logs `decision_made`.

- **Label engine**
  - `computeLabels(xp, completedCount, competencyEvidence)`:
    - Encodes the current “label of excellence” ladder and thresholds.
    - Uses both XP/breadth and the number of competencies with evidence (and how often they appear) to gate higher tiers.
    - Returns an ordered list of label objects (highest tier first), each including an explanation and suggested next steps.
  - `getRank(xp)`:
    - Maintains a coarse-grained level scale that complements labels and acts as a fallback when no labels are awarded.

### Observability & Analytics

- **Event logging** (`logEvent`)
  - Best-effort Firestore writes for key events:
    - `mission_start`, `decision_made`, `mission_complete`.
  - Each event includes:
    - `type`, `payload`, `createdAt`.
  - In the current client-only implementation:
    - **Path**: events are stored under `events/{appId}/users/{uid}/{eventId}`.
    - **Shape**: analytics-friendly fields (scenario id, node, choice, XP deltas, label ids, competency coverage, etc.) live inside `payload`.
    - Workstream E MAY denormalise these payloads into a flatter schema (see v1 schema below) when exporting to a dedicated analytics store.

#### Event schema (v1 – analytics-focused)

Events are stored in Firestore under `events/{appId}/users/{uid}/{eventId}` and are designed to be:

- **Append-only** for auditability.
- **Denormalised** enough to power simple dashboards without joins.
- **Forward-compatible** with the competency and label model from Workstream C.

Each event document SHOULD follow this shape (fields marked with `*` are required):

- `type`* (`string`): high-level category of the event.
  - Examples: `mission_start`, `decision_made`, `mission_complete`, `admin_action`.
- `appId`* (`string`): logical application identifier, used to separate environments/tenants.
- `userId`* (`string`): UID from Firebase Auth.
- `sessionId` (`string`): opaque id that groups events into a single play session.
- `scenarioId` (`string`): id of the active scenario/mission, matching `SCENARIOS[id]`.
- `nodeId` (`string`): id of the narrative node when the event occurred (if applicable).
- `choiceId` (`string`): stable identifier for the selected choice within a node (if applicable).
- `personaId` (`string`): id of the active persona (from `PERSONAS`) when the event was emitted.
- `labelsBefore` (`string[]`): ordered list of label ids before the event.
- `labelsAfter` (`string[]`): ordered list of label ids after the event.
- `xpBefore` (`number`): XP value just before applying this event.
- `xpAfter` (`number`): XP value after applying this event.
- `competencyDeltas` (`Record<string, number>`): map of competency id → delta score, as defined by Workstream C.
- `scenarioTags` (`string[]`): copy of the scenario’s qualitative tags (e.g. domain, difficulty).
- `createdAt`* (`Timestamp`): server-side creation time.
- `clientTime` (`string | Timestamp`): optional client-observed time, for debugging clock skew.
- `version` (`number`): schema version number (initially `1`).

Additional conventions used by the current client implementation:

- For `decision_made` events:
  - `payload.competencyIds` (`string[]`): competency ids inferred from the selected choice’s `tag`, using the Workstream C mapping.
- For `mission_complete` events:
  - `payload.labels` (`string[]`): final label ids (ordered, highest tier first).
  - `payload.competencyEvidenceCoverage` (`number`): count of competencies with non-zero evidence in the user profile.
  - `payload.competencyEvidenceDecisions` (`number`): total number of decision steps that have contributed to competency evidence.

Consumption rules:

- Analytics and dashboards SHOULD treat missing optional fields as `null`/zero rather than failing.
- New fields MUST only be additive and guarded in readers so that historical events remain valid.
- Any competency-related fields (e.g. `competencyDeltas`) MUST reference ids and semantics defined in the Workstream C competency model, without duplicating that model here.

#### Governance, Bias & Compliance (Workstream I)

Workstream I extends observability with governance- and fairness-focused metadata while keeping the v1 event schema **backwards compatible**.

- **Optional top-level fields** (additive, safe to ignore in old readers):
  - `tenantId` (`string`):
    - Copied from `adminConfig.activeTenantId` at event time.
    - Used to partition analytics across organizations; not treated as PII.
  - `cohortIds` (`string[]`):
    - Opaque cohort or archetype identifiers (e.g. training batches, pilot groups).
    - Managed by admins or upstream systems; MUST NOT embed raw PII (emails, employee ids, etc.).
  - `governanceContext` (`{ [key: string]: string | number | boolean }`):
    - Optional small bag of flags for governance (e.g. `{ highRisk: true, scenarioRiskTag: "bias_sensitive" }`).

- **Admin / governance events**:
  - Use `type = "admin_action"` with a more specific subtype in `payload.adminActionType`, for example:
    - Scenario lifecycle: `"scenario_created"`, `"scenario_updated"`, `"scenario_status_changed"`, `"scenario_release_channel_changed"`.
    - Label governance: `"label_awarded_manual"`, `"label_revoked_manual"`, `"label_adjusted_manual"`.
    - Governance configuration: `"governance_policy_updated"`, `"persona_guardrail_updated"`, `"retention_policy_updated"`.
  - Payloads SHOULD include:
    - What changed (previous vs next values).
    - Who changed it (`changedByUserId`, a Firebase UID).
    - Why it changed (`reasonCode`, `freeTextReason`), avoiding PII in free text.

- **Scenario governance metadata (out-of-band from events)**:
  - Stored in Firestore under `governance/{appId}/scenarios/{scenarioId}`.
  - Contains lifecycle status (`draft`, `in_review`, `approved`, `deprecated`), release channel, owner/reviewers, risk tags, and review summaries.
  - Admin Console logic (Workstream D) MUST gate scenario visibility for production tenants based on this metadata.

These additions let Workstream E build governance and fairness dashboards, and let Workstream C/I explain labels and overrides with a clear audit trail, without changing the required core event fields.

### Extensibility and Future Services

- The current client/Firebase implementation is structured so that:
  - `SCENARIOS`, `MODULES`, `PERSONAS`, and label rules are all **data-driven**.
  - A future dedicated backend service (e.g. Node/NestJS) can:
    - Own the scenario definitions and runtime state.
    - Replace `logEvent`/profile persistence with API calls.
    - Expose feature flags and persona catalogs over HTTP.
  - The front-end only needs to swap from direct Firebase usage to a thin API
    client without rewriting the core UX or simulation flows.

#### Multi-tenant readiness (Workstream D)

- **Client-side tenant model**:
  - Admin configuration is expressed as `adminConfig = { activeTenantId, tenants }`.
  - Each tenant entry contains its own `modules`, `enabledScenarioIds`, and `activePersonaId`.
  - The runtime derives an effective tenant config via a helper (e.g. `getTenantConfig(adminConfig)`), which:
    - Falls back to a `"default"` tenant when unset.
    - Merges sparse or legacy configs with safe defaults so missing fields never break the UI.
  - Scenario visibility, persona selection, and module toggles are all resolved **per tenant**.

- **Compatibility with Workstreams C and E**:
  - The competency and label model (Workstream C) remains the single source of truth; tenant config only influences **which** scenarios and personas are available, not how labels or competencies are computed.
  - The event schema (Workstream E) is unchanged; events can optionally be enriched with a `tenantId` field without altering existing readers, as long as it is treated as additive metadata.

- **Migration path to a multi-tenant backend**:
  - **Phase 1 – API façade**:
    - Introduce a thin API layer in front of Firestore that accepts `{ appId, tenantId, userId }` alongside profile and event payloads.
    - Start writing `tenantId` into profile and event documents (e.g. `artifacts/{appId}/users/{uid}/profile` with a `tenantId` field; `events/{appId}/users/{uid}/{eventId}` with `tenantId` in the document body).
  - **Phase 2 – Data re-homing**:
    - Migrate existing documents into tenant-aware collections or projects, for example:
      - Profiles under `tenants/{tenantId}/artifacts/{appId}/users/{uid}/profile`.
      - Events under `tenants/{tenantId}/events/{appId}/users/{uid}/{eventId}`.
    - Keep the original locations readable for a deprecation window so historical dashboards continue to function.
  - **Phase 3 – Backend-owned tenancy**:
    - Move from direct Firestore SDK usage to calling the API exclusively from the client.
    - Let the backend enforce tenant boundaries, per-tenant feature flags, and data retention rules, while the front-end simply passes through `tenantId` derived from `adminConfig.activeTenantId`.

### Workstream K – Experimentation, A/B Testing & Content Calibration

Workstream K introduces a **data-driven experimentation layer** on top of the existing scenario, persona, and module model so that content owners can calibrate difficulty curves and persona behaviour without hardcoding logic into the simulation engine.

#### Experiment configuration model (client-side, data-driven)

Experiments are defined as configuration objects in the client (prototype phase) and are intended to move into the backend `PlatformClient` layer in Workstream J. Each experiment entry follows this shape (conceptual, not a strict TypeScript type):

- `id`* (`string`): stable identifier, e.g. `exp_scenario_vt1_difficulty`.
- `label`* (`string`): human-readable name for admins.
- `description` (`string`): short explanation of the hypothesis being tested.
- `kind`* (`"scenario_variant" | "persona_prompt" | "tooling_bundle"`): what aspect of the experience the experiment targets.
- `targetScenarioIds` (`string[]`): scenario ids this experiment applies to (for `scenario_variant` experiments).
- `targetPersonaIds` (`string[]`): persona ids this experiment applies to (for `persona_prompt` experiments).
- `variants`* (`Array<{ id: string; label: string; weight?: number; }>`):
  - Each variant has:
    - `id`* (`string`): stable variant identifier, e.g. `control`, `treatment_a`.
    - `label`* (`string`): admin-facing label.
    - `weight` (`number`): relative traffic split; defaults to `1` when omitted.
- `assignment` (`{ unit?: "user" | "tenant"; }`):
  - `unit: "user"` → randomised per user id (classic A/B test).
  - `unit: "tenant"` → randomised per tenant id (cohort-based; all users in a tenant share a variant).

In the client-only prototype, experiments are defined as a constant array (e.g. `EXPERIMENTS`) and consumed generically. **No experiment-specific branching is hardcoded into the mission engine**; all routing is driven by configuration.

Examples of early experiments:

- `exp_scenario_vt1_difficulty` (`kind: "scenario_variant"`):
  - Compares a baseline VT1 mission script against a slightly harder variant.
  - `targetScenarioIds: ["vt1"]`.
  - Variants: `control` vs `treatment_a` with equal weights.
  - `assignment.unit: "user"` for individual-level A/B testing.
- `exp_persona_mentor_tone` (`kind: "persona_prompt"`):
  - Compares the baseline mentor persona prompt with a more direct, challenging tone.
  - `targetPersonaIds: ["mentor"]`.
  - `assignment.unit: "tenant"` so that different enterprise tenants experience consistent persona behaviour.

#### Assignment logic and tenant opt-in

Assignment is deterministic and **does not require additional backend state** in the prototype:

- A simple hash of `experiment.id` plus the chosen unit id (user or tenant) is converted into a float in \[0, 1\].
- That value is used to choose a variant according to the configured `weight`s.
- This means a given user (or tenant) will always resolve to the same variant as long as their id and the experiment config remain stable.

Tenant opt-in is handled via the existing `adminConfig` / tenant model:

- Tenant config is extended with an optional `experimentOptInIds: string[]` field.
- The Admin Console exposes an **EXPERIMENT_TRACKS** section where admins can:
  - See all available experiments and their descriptions.
  - Toggle participation per tenant by including or removing experiment ids from `experimentOptInIds`.
- The runtime only activates experiments whose ids are present in the current tenant’s `experimentOptInIds` set.

This approach keeps experiments **data-driven and tenant-aware** without coupling the core mission engine to any specific experiment.

#### Interaction with scenarios, personas, and modules

Experiments affect runtime behaviour in ways that remain **backwards compatible** with existing schemas:

- **Scenarios (`SCENARIOS`)**
  - Scenario variants are modelled as separate scenario entries (e.g. `vt1_control`, `vt1_treatment_a`) or as SDL-imported scenarios.
  - The experiment layer decides which variant id to show to a user and hides the alternative from their scenario list.
  - This is implemented by:
    - Computing active experiments for a given `{ userId, tenantId, scenarioId }` context.
    - Mapping `control` vs `treatment` variants to the appropriate scenario ids.
    - Applying this mapping when building the per-tenant `visibleScenarios` list.
- **Personas (`PERSONAS`)**
  - Persona experiments control which prompt template or persona flavour is used for Gemini calls.
  - The mission engine passes the active experiment variant into the persona selection layer, which chooses an appropriate prompt template, without altering the base `PERSONAS` schema.
- **Modules (`MODULES`)**
  - Experiments can also be defined over module bundles (e.g. enabling `team_mode` for a subset of tenants).
  - In the prototype, this is modelled as configuration only: experiment variants express which module presets to apply; the core module schema and tenant config remain unchanged.

#### Event schema extensions for experimentation (compatible with Workstream E)

To enable Workstream E to compare completion rates, label gains, and feedback across variants, the event schema is **extended in an additive way**:

- Events may include an optional `tenantId` field (as already anticipated in this document).
- Events may include an optional `experiments` field:
  - Shape: `Record<string, string>` mapping `experimentId → variantId` that were active when the event was emitted.
  - Example:
    - `experiments: { "exp_scenario_vt1_difficulty": "treatment_a", "exp_persona_mentor_tone": "baseline" }`.
- A dedicated `experiment_assignment` event type is introduced:
  - Emitted when a mission is started and experiment variants are resolved.
  - Fields:
    - `type: "experiment_assignment"`.
    - `scenarioId` (if applicable).
    - `tenantId` (if available).
    - `experiments` map as above.

These additions are **purely additive**:

- Existing readers that ignore `tenantId` and `experiments` continue to work.
- New analytics views can group by `experiments[experimentId]` to compare funnels, XP/label deltas, and any future subjective feedback metrics between variants.

### Workstream J – API Boundary, Strangler Pattern, and Performance Plan

#### Frontend vs API layer boundary

- **Frontend (console + tools)**
  - Renders missions, personas, analytics views, and admin tooling.
  - Maintains **local simulation state** for the current mission (current node, history, transient UI-only flags).
  - Interacts with a single abstraction: `PlatformClient`, instead of directly calling Firebase SDKs or Gemini.

- **API layer (modular monolith backend)**
  - Owns **mission progression**, **profile persistence**, **label computation orchestration**, and **event logging**.
  - May run as a Node/TypeScript app (e.g. Express/Nest-like) and talk to Firebase, Gemini, and future data stores.
  - Treats Workstream C’s competency and label models and Workstream I’s governance rules as **authoritative inputs**, not something it redefines.

#### Service contracts (v1, language-agnostic)

These contracts are deliberately small and map to existing in-browser helpers; they are meant to be implemented either:

- Directly in the browser (compatibility mode, using Firebase and Gemini SDKs), or
- On the backend (API mode), with the frontend calling over HTTP/JSON or RPC.

Conceptually:

- **Mission / simulation**
  - `startMission(request)`:
    - Input (required): `appId`, `userId`, `scenarioId`.
    - Input (optional): `tenantId`, `sessionId`, `personaId`.
    - Output: `sessionId`, initial `node` (scene + choices), current `profile` (see Game State), and any pre-computed labels.
  - `submitDecision(request)`:
    - Input: `appId`, `userId`, `scenarioId`, `sessionId`, `nodeId`, `choiceId`, optional `tenantId` and `personaId`.
    - Output: next `node` (if not terminal), updated `profile`, updated labels, and any structured metadata needed by Workstreams C/E (e.g. `competencyDeltas`, `labelsBefore/After`).

- **Profile**
  - `getProfile(request)`:
    - Input: `appId`, `userId`, optional `tenantId`.
    - Output: the profile document (same shape as `artifacts/{appId}/users/{uid}/profile`).
  - `updateProfile(request)`:
    - Input: `appId`, `userId`, optional `tenantId`, and a patch compatible with the existing profile schema.
    - Output: updated profile.

- **Events**
  - `logEvent(request)`:
    - Input: any event following the v1 schema (type, appId, userId, scenarioId, etc.), plus optional extensions such as `tenantId` or tool-related fields.
    - Output: acknowledgement only; consumers are expected to read from Firestore or a future analytics store.

- **Persona / Gemini**
  - `getMentorFeedback(request)`:
    - Input: scenario and decision context, persona id, and relevant parts of the user profile.
    - Output: persona-conditioned feedback text, optional TTS URL or metadata, and structured scores where applicable.

In TypeScript, this can be expressed as a shared interface without using `any`. For example:

```ts
export interface PlatformClient {
  startMission(request: StartMissionRequest): Promise<StartMissionResponse>;
  submitDecision(request: SubmitDecisionRequest): Promise<SubmitDecisionResponse>;
  getProfile(request: ProfileRequest): Promise<ProfileDocument>;
  updateProfile(request: UpdateProfileRequest): Promise<ProfileDocument>;
  logEvent(event: AnalyticsEvent): Promise<void>;
  getMentorFeedback(request: MentorFeedbackRequest): Promise<MentorFeedbackResponse>;
}
```

- `ProfileDocument`, `AnalyticsEvent`, and the various request/response types are thin wrappers around the existing profile and event shapes described above, with only **additive optional fields** for tenancy, tooling, and performance metadata.

#### Strangler pattern & compatibility modes

- **Step 1 – In-browser façade**
  - Introduce a `PlatformClient` implementation inside `index.html` that:
    - Wraps all Firebase Auth, Firestore profile, Firestore event, and Gemini calls.
    - Exposes only the methods listed above to the rest of the React app and simulation engine.
  - Existing helpers (`startSim`, `handleDecision`, `logEvent`, label computation) are refactored to depend on this `PlatformClient` instead of direct SDK calls.
  - A configuration flag (e.g. `platformMode: "firebase" | "api"`) is introduced, defaulting to `"firebase"` so the current UX stays intact.

- **Step 2 – Optional Node/TypeScript backend**
  - Implement a backend service that:
    - Exposes HTTP endpoints mirroring the `PlatformClient` methods (e.g. `POST /api/v1/missions/start`, `POST /api/v1/missions/decision`, `GET /api/v1/profile`).
    - Internally uses the same profile and event schema (including Workstream C’s competency fields and Workstream E’s event model).
    - Applies governance, rate limiting, and logging standards defined by Workstream I.
  - The frontend `PlatformClient` gains a second implementation that:
    - Calls these endpoints using `fetch`.
    - Preserves the same request/response types so the rest of the app is unaware of the transport.
  - Rollout is gradual:
    - Start with low-risk paths (e.g. analytics event logging), then profile reads/writes, then mission progression and Gemini orchestration.

#### Event extensions for tools and performance (additive only)

- **New event type**: `tool_invoked`
  - Fits into the existing `type` field of the v1 schema (no structural change).
  - Intended for admin actions, console tools, and future internal utilities that should be observable.
  - Recommended payload fields (all optional and additive):
    - `payload.toolId` (`string`): stable identifier for the tool (e.g. `admin.toggleModule`, `analytics.missionFunnel`).
    - `payload.toolCategory` (`string`): high-level grouping (e.g. `admin`, `analytics`, `simulation_debug`).
    - `payload.durationMs` (`number`): client- or server-measured duration of the tool action.
    - `payload.status` (`"success" | "error" | "cancelled"`).
    - `payload.errorCode` (`string`): short code for failures, if any.
  - These are designed so that:
    - Existing consumers that ignore `tool_invoked` events or unknown payload fields continue to work.
    - Workstreams E and I can build analytics and governance views without needing a separate event stream.

- **Performance metadata (optional fields)**
  - Gemini-related events (e.g. `decision_made`, `mission_complete`, and `tool_invoked` for mentor tools) MAY include:
    - `payload.geminiModel` (`string`): model identifier, e.g. `gemini-2.5-flash`.
    - `payload.geminiLatencyMs` (`number`).
    - `payload.geminiInputTokens` / `payload.geminiOutputTokens` (`number`).
  - Backend-served events MAY include:
    - `payload.backendLatencyMs` (`number`).
    - `payload.backendInstanceId` (`string`): opaque id for debugging.
  - All of these fields are **optional** and must be treated as best-effort hints by readers.

#### Performance profiling & Gemini optimisation

- **Hot spots to measure**
  - Gemini:
    - Per-decision mentor feedback calls.
    - Optional TTS generation for scenes and summaries.
  - Rendering:
    - SVG pathway updates and mission views on low-powered devices.
  - Firestore:
    - Profile reads at login / mission start.
    - Event writes for every decision.

- **Strategies (client and backend)**
  - **Budgeted Gemini calls**:
    - Prefer fewer, higher-value calls (e.g. summary feedback at key decision points) over many micro-calls.
    - Cache deterministic feedback where the input is stable: same `scenarioId`, `nodeId`, `choiceId`, and `personaId`.
  - **Firestore I/O shaping**:
    - Keep profile reads to a minimum (load once per session, then rely on in-memory state).
    - Buffer non-critical events in memory and write in small batches where it does not risk data loss or UX regressions.
  - **Rendering hygiene**:
    - Ensure mission components subscribe only to the state they actually need.
    - Defer heavy analytics queries to background views or server-side aggregation in the backend.

These changes are **additive** to the existing architecture: the current single-page console continues to work in compatibility mode, while Workstream J introduces the contracts and performance hooks needed for a gradual evolution toward a modular monolith backend.