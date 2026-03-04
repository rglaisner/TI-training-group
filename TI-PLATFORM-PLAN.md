## TI Certification Simulation Platform – Master Plan

This document is the high-level, human-readable plan for the TI Certification Simulation Platform. It is intended as a **single source of truth** that other agents and collaborators can reference.

---

### 1. Product & Experience Vision

- **Purpose**: Provide a **living, RPG-like simulation** where Talent Intelligence (TI), Workforce Planning, and People Analytics professionals can demonstrate and maintain **labels of excellence**, rather than pass a one-off exam.
- **Experience principles**:
  - **UI/UX first**: console-like interface, clear navigation, and no dead ends.
  - **Realistic scenarios**: messy stakeholders, incomplete data, time pressure, competing priorities.
  - **Continuous assessment**: labels are maintained (or decayed) based on ongoing performance.
  - **Personas & teams**: simulated colleagues, managers, and mentors with distinct, human-like reactions.
  - **Multimodal**: text, interactive data views, and (optionally) voice feedback via Gemini.
  - **Enterprise-grade direction**: modular, observable, and ready to grow beyond the current prototype.

---

### 2. Current Architecture Snapshot

See `[ARCHITECTURE.md](ARCHITECTURE.md)` for full detail. At a glance:

- **Client-first web app** (`index.html`):
  - React UI (UMD) + TailwindCSS.
  - Simulation logic (scenarios, decisions, XP, labels) lives in the browser.
- **Backend via Firebase**:
  - Anonymous **Firebase Auth** for identity.
  - **Firestore** for:
    - Profiles: XP, completed scenarios, labels.
    - Events: mission start, decisions, mission completion.
- **Gemini integration**:
  - Mentor / persona feedback on decisions.
  - Optional text-to-speech narration.
- **Data-driven model**:
  - `SCENARIOS` define missions and nodes.
  - `MODULES` and `adminConfig` implement feature flags.
  - `PERSONAS` shape the tone and perspective of feedback.

---

### 3. Epic Checklist (Tick-Box Progress)

Use this section to track major workstreams. Tick boxes reflect **current** implementation status.

#### 3.1 Product & UX Foundations

- [x] Define labels-of-excellence concept and continuous validation idea.
- [x] Implement the TI console shell UI (header, nav, main views).
- [ ] Add richer onboarding and calibration missions for new users.
- [ ] Introduce narrative explanation pages for labels and competencies.

#### 3.2 Simulation Engine & Scenarios

- [x] Implement SDL-like scenarios (nodes, choices, XP, tags) in `index.html`.
- [x] Persist XP and completion per user profile.
- [x] Add a multi-stage squad-based mission (`vt1` – Virtual War Room).
- [x] Support multi-mission arcs with cross-mission state (reputation, risk).
- [x] Implement time-delayed consequences and simulated time progression.

#### 3.3 Personas & Social Simulation

- [x] Implement persona configuration (`PERSONAS` for mentor, boss, peer).
- [x] Condition Gemini feedback on the active persona in Admin Console.
- [x] Allow multiple personas to participate in the same interaction step.
- [x] Persist persona “memory” across missions (trust, history).

#### 3.4 Admin & Modularity

- [x] Admin Console UI with module toggles (`MODULES` + `adminConfig.modules`).
- [x] Scenario visibility filter (`enabledScenarioIds` + `getVisibleScenarios`).
- [x] Introduce tenant-/cohort-based configuration (per-organization settings).
- [x] Add admin tooling for scenario pack management and release channels.

#### 3.5 Data & Observability

- [x] Event logging for mission start, decision, and completion (`logEvent`).
- [x] Define a unified event schema (v1) for analytics (see `ARCHITECTURE.md`).
- [ ] Build internal dashboards or pages to visualize:
  - [ ] Scenario usage and completion rates.
  - [ ] Drop-off points and label distribution.
- [ ] Add simple health and cost metrics for Gemini usage.

#### 3.6 Enterprise Readiness

- [ ] Define an auth strategy beyond anonymous Firebase (e.g., email, SSO).
- [ ] Design data retention, export, and privacy policies for simulation data.
- [ ] Plan migration from Firebase-only backend toward a modular API layer.

---

### 4. Risks & Trade-offs

- **Client-heavy architecture**:
  - + Fast to iterate, minimal infra.
  - − Harder to enforce enterprise security and multi-tenant boundaries.
- **Firebase coupling**:
  - + Simple auth and storage to start.
  - − Migrating organizations or self-hosted deployments will require an extraction step.
- **Gemini free-tier constraint**:
  - + No direct model costs.
  - − Requires careful prompt design and call frequency management to stay within limits.
- **Single-page prototype**:
  - + Easy to reason about and demo.
  - − Will eventually need to be broken into a more maintainable, testable structure.

---

### 5. Ongoing Sub-Plan (Workstreams)

This section outlines how to evolve the prototype into the fuller platform. Each workstream can later become its own `.plan.md`.

#### Workstream A – Scenario & Simulation Depth

- Expand from single-session scenarios into richer missions with:
  - Time progression (simulated weeks/months) and delayed outcomes.
  - Shared state across missions (e.g., persona trust, organizational risk).
- Add support for in-simulation tools:
  - Market and capacity dashboards.
  - Simple what-if calculators for headcount and locations.

#### Workstream B – Persona & Team Dynamics

- Extend personas so:
  - Boss and peer can both react and sometimes conflict in a single step.
  - Personas can reference concrete past missions and decisions.
- Design and implement at least one **team-mode mission** where:
  - The user leads an AI squad through a multi-step challenge.
  - The storyline emphasizes alignment, disagreement, and escalation.

#### Workstream C – Certification Logic & Competencies

- [x] Formalise a competency framework for TI, Workforce Planning, and People Analytics.
- [x] Link scenario tags and decision patterns to competencies.
- [x] Evolve labels of excellence from pure XP thresholds to:
  - [x] Evidence-based competency coverage.
  - [x] Minimum breadth across scenario types (XP + completed missions gating).
- [x] Provide transparent explanations of:
  - [x] Why a user holds a label.
  - [x] What evidence supports it.
  - [x] What to do next to grow.
- [ ] Calibrate thresholds and competency mappings using real user data from analytics (iterative).

#### Workstream D – Admin & Multi-Tenant Readiness

- Generalise Admin Console to:
  - Manage tenants/cohorts and assign scenario packs.
  - Configure personas and modules per tenant.
- Design a migration from single-tenant Firebase to:
  - Either a multi-tenant API backend.
  - Or per-tenant isolated Firebase projects, if appropriate.

#### Workstream E – Observability & Analytics UX

- Build simple internal analytics views for:
  - Scenario popularity and completion.
  - Drop-off steps and average XP/labels over time.
- Use these insights to:
  - Calibrate scenario difficulty.
  - Identify content gaps and over-used patterns.

Initial scope for Workstream E:

- **Event model alignment**
  - Adopt the v1 event schema from `ARCHITECTURE.md` as the contract for analytics.
  - Ensure fields needed by the competency/label model (Workstream C) are present but optional (e.g. `competencyDeltas`, `labelsBefore/After`).
- **Analytics views (first iteration)**
  - Console-only internal views, backed directly by Firestore `events` + `profiles`:
    - Mission funnel view: missions started vs completed, drop-off by node.
    - Scenario catalog view: popularity, completion rate, average XP delta per scenario.
    - Label distribution view: how labels change over time across the user base.
  - Keep queries denormalised and simple to avoid premature backend complexity.
- **Feedback loop into content**
  - Define a lightweight process where scenario owners review analytics (e.g. monthly) to:
    - Adjust scenario difficulty and XP rewards.
    - Identify missing scenario types or over-used patterns.
  - Document how Workstream C can use the same views to validate competency coverage.
﻿
#### Workstream K – Experimentation, A/B Testing & Content Calibration

- **Goal**
  - Build the ability to run controlled experiments on scenarios, difficulty curves, personas, and tool bundles so that content owners can calibrate the experience using real player data.

- **Scope**
  - **Experiment model (data-driven)**
    - Define a first-class experiment configuration model (see `ARCHITECTURE.md` → *Workstream K – Experimentation, A/B Testing & Content Calibration*):
      - Experiments are rows in a config array (e.g. `EXPERIMENTS`) rather than hard-coded `if` statements.
      - Each experiment declares `id`, `label`, `description`, `kind`, `variants`, and an `assignment` unit (`user` vs `tenant`).
      - Early experiment types:
        - Scenario variants (e.g. VT1 difficulty curves, alternate node scripts).
        - Persona prompt variants (e.g. mentor tone v1 vs v2).
        - Tooling bundles (e.g. enabling `team_mode` or certain tools for a subset of tenants).
  - **Assignment logic (random vs cohort-based)**
    - Deterministic, hash-based assignment that does not require additional backend state:
      - `unit: "user"` → per-user randomisation for classic A/B tests.
      - `unit: "tenant"` → per-tenant randomisation so cohorts/enterprises share a consistent variant.
    - Ensure assignments are **sticky** across sessions given the same ids and config.
  - **Tenant / module integration**
    - Extend tenant config with an additive `experimentOptInIds: string[]` field.
    - Add an **EXPERIMENT_TRACKS** panel to the Admin Console to:
      - List available experiments with short descriptions.
      - Allow admins to opt specific tenants in or out by toggling membership in `experimentOptInIds`.
    - Keep module and persona schemas unchanged; experiments only influence which scenarios, personas, or module presets are active.
  - **Observability extensions (building on Workstream E)**
    - Extend the v1 event schema **additively**:
      - Add optional top-level `tenantId` (already anticipated in `ARCHITECTURE.md`) to all mission events.
      - Add optional top-level `experiments: Record<string, string>` mapping `experimentId → variantId` active at event time.
      - Introduce a dedicated `experiment_assignment` event type emitted when a mission starts.
    - Ensure all existing event consumers treat missing `experiments` as "no experiment" rather than as an error.
    - Document how analytics views (Workstream E) can:
      - Compare mission funnels, XP deltas, and label transitions by `experiments[experimentId]`.
      - Later incorporate subjective feedback signals (e.g. player ratings, NPS-style questions) without changing the core experiment shape.

- **Dependencies**
  - **E – Observability & Analytics UX**: provides the base event schema and analytics views that K will stratify by variant.
  - **C – Certification Logic & Competencies**: competencies and labels remain the canonical definitions; experiments focus on *how* content is presented, not on changing competency semantics.
  - **D – Admin & Multi-Tenant Readiness**: supplies the tenant model into which experiment opt-ins are wired.

#### Workstream I – Governance, Bias & Compliance Guardrails

See `GOVERNANCE-BIAS-COMPLIANCE.md` for the detailed governance spec used by all workstreams.

- **Goal**: Make the platform trustworthy for enterprise use through clear governance over scenarios, AI output, fairness, and compliance.
- **Scope**:
  - Define scenario lifecycle and governance metadata:
    - Scenario review states (`draft`, `in_review`, `approved`, `deprecated`) stored separately from scenario content.
    - Scenario governance documents in Firestore (`governance/{appId}/scenarios/{scenarioId}`) including owner, reviewers, risk tags, and review summaries.
    - Admin Console gates that only allow `approved` scenarios to be enabled for production tenants.
  - Strengthen label governance:
    - Treat `labelsBefore/After` in events as the canonical audit trail for automatic label changes.
    - Define manual label override flows (award/revoke/adjust) and require append-only audit events with reasons and admin identifiers.
  - Introduce bias and fairness guardrails:
    - Shared Gemini persona guardrail preamble (non-discrimination, no PII, focus on decision quality).
    - Scenario review checklists for bias and ethical concerns during `in_review → approved` transitions.
    - Event and profile fields for non-PII tenant and cohort metadata to support fairness analytics.
  - Outline privacy & retention policies:
    - No storage of raw PII in profiles, events, or prompts.
    - Directional policies for retention, aggregation, and export, to be implemented per tenant in Enterprise Readiness work.
- **Dependencies**:
  - Workstream C (Certification Logic & Competencies) for label computation and competency evidence semantics.
  - Workstream D (Admin & Multi-Tenant Readiness) for tenant-aware governance UI, retention settings, and cohort management.
  - Workstream E (Observability & Analytics UX) for governance and fairness dashboards built on the extended event schema.

#### Workstream J – Performance, Scaling & Architecture Evolution

- **Goal**
  - Prepare the console-first Firebase prototype to scale into a modular, segmentable architecture **without breaking the current UX**.

- **Boundary: Frontend vs API layer**
  - Frontend (console, tools):
    - Owns rendering, local mission state, and lightweight orchestration.
    - Talks only to a thin **platform client** (`PlatformClient`) instead of directly to Firebase or Gemini.
  - API layer (simulation engine, personas, profiles, events):
    - Owns mission progression, profile mutations, label computation orchestration, and event recording.
    - Exposes a small set of HTTP/JSON endpoints or RPC-style methods (e.g. `startMission`, `submitDecision`, `getProfile`, `logEvent`) that mirror the existing in-browser helpers.
    - Treats competency and label definitions from Workstream C as **read-only inputs**.

- **Migration plan (strangler pattern)**
  - Step 1 – Wrap current Firebase and Gemini usage:
    - Introduce a `PlatformClient` in the frontend that wraps:
      - `ProfileService` (Firestore profile reads/writes).
      - `EventService` (Firestore event writes).
      - `GeminiPersonaService` (mentor feedback + optional TTS).
    - Keep implementation in **compatibility mode** (`mode: "firebase"`) so `index.html` continues to function with no backend.
  - Step 2 – Optional Node/TypeScript backend:
    - Implement the same `PlatformClient` contract on a Node/TypeScript modular monolith that:
      - Talks to Firebase (or a future data store) and Gemini on behalf of the client.
      - Follows the v1 event schema from `ARCHITECTURE.md` and Workstream C’s competency model.
    - Allow the frontend to switch between `mode: "firebase"` and `mode: "api"` via configuration, so rollout is gradual and reversible.

- **Performance & Gemini constraints**
  - Hot spots to monitor:
    - Gemini: per-decision mentor feedback and TTS calls.
    - Rendering: re-renders of mission views and SVG pathway.
    - Firestore I/O: profile reads/writes and per-step event logging.
  - Optimisation strategies:
    - Introduce **structured logging** of latency, token usage, and error rates for each Gemini and Firestore call.
    - Add **caching** for deterministic Gemini responses (e.g. same scenario node, choice, persona) at the API layer, keyed by scenario + node + choice + persona.
    - Batch non-critical Firestore writes (e.g. buffer low-priority events during a mission and flush at `mission_complete`) where UX is not impacted.
    - Ensure every user-facing tool action emits a `tool_invoked` event (see `ARCHITECTURE.md`) to support analytics and governance (Workstreams E and I).

- **Coordination with other workstreams**
  - Workstreams F, G, H, and J:
    - Treat the `PlatformClient` contract and event schema as shared infrastructure; avoid duplicating connectivity logic.
  - Workstreams C and I:
    - Remain the single source of truth for competencies, labels, and governance rules; the API layer should not embed its own scoring or policy logic.
  - Breaking changes:
    - Any future change to mission, profile, or event contracts should be expressed as **additive optional fields** and first documented here and in `ARCHITECTURE.md` before implementation.