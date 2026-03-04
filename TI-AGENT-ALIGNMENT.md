## TI Agent Alignment & Cross-Workstream Contracts

This document is the **single alignment layer** for all specialised agents working on the TI Certification Simulation Platform across Workstreams A–K.

- **Audience**: agent prompts (F, H, I, J, K, etc.), human maintainers.
- **Sources of truth**: this doc MUST stay consistent with:
  - `TI-PLATFORM-PLAN.md`
  - `ARCHITECTURE.md`
  - `GOVERNANCE-BIAS-COMPLIANCE.md`
  - `README.md`

If you believe this document conflicts with those, **stop and report the conflict** instead of inventing new conventions.

---

### 1. Non‑negotiable invariants

- **Event schema v1 is canonical**  
  - Shape and semantics live in `ARCHITECTURE.md`.  
  - All extensions must be **additive and optional**; old readers must continue to work unchanged.

- **Competency & label model is canonical (Workstream C)**  
  - No workstream may redefine competency ids, label semantics, or scoring rules.  
  - SDL, tools, governance, and experiments may only emit **signals** that C/E interpret.

- **Governance rules are canonical (Workstream I)**  
  - Scenario/persona lifecycle, bias guardrails, and audit rules live in `GOVERNANCE-BIAS-COMPLIANCE.md`.  
  - Other workstreams consume governance metadata; they do not own governance logic.

- **PlatformClient is the runtime boundary (Workstream J)**  
  - All new runtime work must go through `PlatformClient` methods, not direct Firebase/Gemini calls.  
  - Two modes only: `firebase` (client-only façade) and `api` (HTTP backend), sharing the same interface.

---

### 2. Cross‑workstream contracts

#### 2.1 Scenarios, SDL, and experiments (F ↔ C ↔ K)

- **Scenario ownership**
  - Agent F’s `MissionSDL` is the authoring format for missions; loaders compile SDL into the runtime `SCENARIOS`/`nodes` shape in `index.html`.
  - Each mission has a stable, human‑readable `scenarioId`.

- **Experiments and variants**
  - Experiments (Workstream K) are **configuration** that select among whole missions.  
  - **Decision**: Scenario variants are separate SDL missions (e.g. `vt1_control`, `vt1_treatment_a`); experiments choose which id is visible.
  - **Contract**:
    - All SDL missions must be addressable via stable `scenarioId`s.
    - Experiments never alter SDL semantics; they only route players to different mission ids.

- **SDL ↔ competencies & labels**
  - SDL may attach decision `tag`s and optional `skillSignals`, but the mapping to competencies/labels flows through:
    - `TAG_TO_COMPETENCY_IDS`
    - `COMPETENCIES`
    - `computeLabels` and `competencyEvidence` in `ARCHITECTURE.md`
  - SDL must not define its own competency or label models.

#### 2.2 SDL, external data, and tools (F ↔ H)

- **Where external data lives**
  - `TI-PLATFORM-PLAN.md` specifies that the Market & Comp Visualizer is driven by `externalData.requests` on the **current node**.
  - **Decision**: Node‑level external data wiring is part of the **scenario/SDL layer**, not a separate tool‑only config.

- **Contracts**
  - SDL (`StageSDL`) exposes an optional `externalData`/`externalRequests` block per stage/node describing which external feeds the node requires.
  - Tools (Headcount Planner, Market & Comp Visualizer, Evidence Board, etc.):
    - Are registered in a global `TOOL_DEFINITIONS` catalog.
    - Are enabled via scenario‐level and node‑level config compiled from SDL into `ScenarioToolsConfig`.
    - **Consume** node‑level external data references when building their views; they do not own those references.

#### 2.3 Tools, governance, and risk (H ↔ I)

- **High‑risk scenarios**
  - Governance docs introduce `riskTags` (e.g. `bias_sensitive`, `pii_adjacent`, `high_impact_label`) and `governanceContext`.
  - **Decision**: `isHighRiskScenario` is derived from governance metadata, not hard‑coded into SDL or tools.

- **Contracts**
  - A shared helper (implemented in the runtime/API layer) exposes governance state:
    - `getScenarioGovernance(scenarioId) → { governanceStatus, releaseChannel, riskTags, ... }`
    - `isHighRiskScenario(scenarioId) → boolean`
  - Agent H’s `getToolsForScenario`:
    - Accepts a **boolean flag** (and/or simple enum) such as `isHighRiskScenario`.
    - **Does not interpret raw `riskTags` itself.**
  - Governance (I) also owns:
    - Tenant‑level `TenantToolPolicy` (which tools are disabled/required per tenant).
    - Bias‑related governance flags consumed by tools (e.g. high‑risk scenarios must offer `evidence_board`).

#### 2.4 Governance lifecycle and UX surfacing (F ↔ I ↔ D)

- **Lifecycle is admin‑only**
  - `governanceStatus` (`draft`, `in_review`, `approved`, `deprecated`) and `releaseChannel` (`experimental`, `beta`, `ga`) are stored in governance docs, not in SDL.
  - **Decision**:
    - `governanceStatus` is an **admin‑only** concept used to gate which scenarios/personas can be enabled for tenants.
    - `releaseChannel` may be surfaced in admin tooling or as a subtle badge (e.g. “Experimental”) but is never used to bypass governance.

- **Contracts**
  - SDL may include **read‑only pointers** to governance docs (e.g. `metadata.governanceRefId`) but must not encode lifecycle logic.
  - Runtime reads canonical lifecycle status from Firestore under `governance/{appId}/scenarios/{scenarioId}` and gates visibility accordingly.

#### 2.5 PlatformClient, events, and additive fields (J ↔ E ↔ I ↔ K)

- **PlatformClient boundary**
  - All mission progression, profile persistence, event logging, and mentor feedback flow through:
    - `startMission`
    - `submitDecision`
    - `getProfile`
    - `updateProfile`
    - `logEvent`
    - `getMentorFeedback`
  - Implementations:
    - `firebase` mode: in‑browser façade wrapping Firebase+Gemini.
    - `api` mode: HTTP client calling a Node/TypeScript backend with the same types.

- **Event extensions**
  - Optional, additive top‑level fields (per `ARCHITECTURE.md` and governance spec):
    - `tenantId?: string`
    - `cohortIds?: string[]`
    - `governanceContext?: Record<string, string | number | boolean>`
    - `experiments?: Record<string, string>` (experimentId → variantId)
    - Performance metadata (e.g. `geminiModel`, `geminiLatencyMs`, token counts, `backendLatencyMs`, `backendInstanceId`)
  - **Decision**:
    - Readers must treat missing values as “unknown” or neutral.
    - No existing required fields are removed or re‑typed.

- **Experiments**
  - Workstream K:
    - Owns `EXPERIMENTS` config plus deterministic assignment.
    - Writes `experiments` maps into events and passes them into PlatformClient calls.
  - Experiments never change label or competency semantics; they only affect which scenarios, personas, or tool bundles are active.

---

### 3. Resolved answers to prior agent questions

This section closes the main open questions raised by earlier agents.

#### 3.1 Agent F – Scenario Authoring & SDL Tooling

- **Are scenario variants separate missions?**  
  Yes. Variants (e.g. `vt1_control`, `vt1_treatment_a`) are distinct SDL missions. Experiments decide which variant a player sees.

- **Where should node‑level external data for tools live?**  
  In SDL, as node‑level `externalData`/`externalRequests` on `StageSDL`. Tools read these via the runtime nodes; no separate tool‑only config is introduced for this.

- **Should governance status be visible to players?**  
  No. Governance status is admin‑only. Optional `releaseChannel` badges may be surfaced, but lifecycle states (`draft`, `in_review`, etc.) remain in admin/governance views only.

- **Will a future API‑backed engine share the same SDL/runtime contracts?**  
  Yes. `MissionSDL` and `RuntimeScenario/RuntimeNode` are shared contracts between frontend and backend. Validation and loading can move server‑side without changing their shapes.

#### 3.2 Agent H – Advanced Multimodality & Tools Workspace

- **How do tools discover external data?**  
  Through node‑level `externalData` defined in SDL and compiled into runtime nodes. Tools never hard‑code vendor feeds; they consume abstract data keys.

- **Who decides whether a scenario is “high risk”?**  
  Governance (I), via `riskTags` and governance docs. Tools only receive a derived boolean/enum (e.g. `isHighRiskScenario`) from a shared helper.

- **Who owns tenant‑specific tool policies?**  
  Governance/Admin (I/D) own `TenantToolPolicy`. Tools interpret that policy but do not write it.

#### 3.3 Agent I – Governance, Bias & Compliance

- **What vocabularies and SLAs apply for MVP?**
  - Starter `riskTags`: `bias_sensitive`, `pii_adjacent`, `high_impact_label`.  
  - Starter `reasonCode`: `external_evidence`, `bug_fix`, `appeal_upheld`, `tenant_request`.  
  - Review SLAs: any scenario/persona with sensitive `riskTags` must be re‑reviewed when:
    - Its content changes substantially, or
    - The competency/label framework changes in a way that could alter outcomes.

- **How far does automated bias checking go under Gemini free‑tier?**
  - MVP automation is **offline and sampled**:
    - Batch review a subset of scenarios/decisions using Gemini, not per‑player, per‑event checks.
    - Findings are stored as governance notes or `admin_action` events, not shown directly to players.

#### 3.4 Agent J – Performance, Scaling & Architecture Evolution

- **Are additive event fields for tenancy/experiments/governance allowed?**  
  Yes, provided they are strictly optional and old readers ignore them safely. This includes `tenantId`, `cohortIds`, `governanceContext`, `experiments`, and performance metadata.

- **Can PlatformClient be implemented both in‑browser and on the backend without changing callers?**  
  Yes. All contracts are designed so callers are agnostic to `mode: "firebase" | "api"`. Switching modes is controlled by configuration, not by API shape.

---

### 4. Still‑open topics (do not hard‑code)

The following areas are intentionally left flexible and must **not** be locked into code without an explicit update to this doc and the core specs:

- Exact, exhaustive lists of:
  - `riskTags`
  - `reasonCode`
  - Detailed bias/fairness checklist questions
- Detailed analytics mappings from tool events to competency deltas or labels:
  - These live in Workstreams C/E configuration, not in runtime code.
- Deeper experimentation analytics:
  - How specific metrics are sliced by experiments beyond the basic `experiments[experimentId]` grouping.

When in doubt, emit **structured, additive signals** and let analytics/governance layers interpret them.

---

### 5. How agents must use this document

Any new or rerun agent prompt that touches scenarios, tools, governance, events, or PlatformClient MUST:

1. **Read this document first**, then the core specs:
   - `TI-AGENT-ALIGNMENT.md`
   - `TI-PLATFORM-PLAN.md`
   - `ARCHITECTURE.md`
   - `GOVERNANCE-BIAS-COMPLIANCE.md`
   - `README.md`
2. **Treat this doc as binding** unless it clearly contradicts the core specs.
3. When discovering a conflict:
   - Stop after summarising the conflict.
   - Do not change code/specs to “work around” it.
   - Hand the conflict back to the human maintainer (and, if appropriate, update this doc after agreement).

---

### 6. Ready‑to‑use prompt snippets (F, H, I, J)

You can paste these blocks directly into new agent prompts. Adjust wording minimally if needed, but keep the dependencies and pause/play rules.

#### 6.1 Agent F – SDL alignment & externalData wiring

> You are **Agent F**, focused on **Scenario Authoring & SDL Tooling** for the TI Certification Simulation Platform.  
>  
> **Read these first, in order:**  
> - `TI-AGENT-ALIGNMENT.md` – especially the cross‑workstream contracts and the resolved F/H questions.  
> - `TI-PLATFORM-PLAN.md` – Simulation Engine & Scenarios; Admin & Modularity; Workstreams A/F/H.  
> - `ARCHITECTURE.md` – Core Domain Objects, Simulation Engine, Competency Catalog, Label of Excellence, Workstream H & K notes.  
> - `GOVERNANCE-BIAS-COMPLIANCE.md` – how governance metadata is stored and used.  
> - `README.md` – Platform Data Flow and Functional Simulation Flow.  
>  
> **Constraints:**  
> - The runtime/event schema in `ARCHITECTURE.md` is authoritative; any new fields must be **additive and optional**.  
> - Experiments (Workstream K) select among whole missions, not partial nodes.  
> - Governance lifecycle (`governanceStatus`) is admin‑only; SDL may hold read‑only references but must not encode lifecycle logic.  
>  
> **Tasks (in order):**  
> 1. Reconcile the SDL spec with tools & external data: ensure each `StageSDL` can optionally declare `externalData.requests` compatible with the Market & Comp Visualizer; confirm `tools.enabledToolIds` and per‑node overrides compile cleanly into `ScenarioToolsConfig`. Stop after documenting remaining ambiguities; do **not** assume code has been changed.  
> 2. Define the minimal runtime additions (e.g. `node.externalData`, `node.toolsConfig`) needed to consume SDL and feed tools without breaking existing missions. Document how validation errors involving external data or tools are surfaced to authors vs players.  
> 3. *(Optional, planning only)* Sketch how one existing hard‑coded mission could be migrated to SDL and how you would test equivalence.  
>  
> **Pause/play:** If you detect any conflict between `TI-AGENT-ALIGNMENT.md` and the base specs, stop and report the conflict instead of inventing new conventions.

#### 6.2 Agent H – Tools workspace implementation prep

> You are **Agent H**, focused on **Advanced Multimodality & Tools Workspace** for the TI Certification Simulation Platform.  
>  
> **Read these first, in order:**  
> - `TI-AGENT-ALIGNMENT.md` – especially tools/governance and the `isHighRiskScenario` contract.  
> - `TI-PLATFORM-PLAN.md` – UX & Interaction Design, Simulation Workspace, Tools Sidebar, Workstream H.  
> - `ARCHITECTURE.md` – Simulation Engine, Game State/meta, Event schema (v1), Workstreams H and K.  
> - `GOVERNANCE-BIAS-COMPLIANCE.md` – risk tags, governanceContext, bias/fairness guardrails.  
> - `README.md` – how the current mission view behaves.  
>  
> **Tasks (in order):**  
> 1. Align `TOOL_DEFINITIONS`, `ScenarioToolsConfig`, and `TenantToolPolicy` with the final SDL design where SDL owns `tools.enabledToolIds` and node‑level `externalData.requests`. Recommend only **additive** tweaks so loaders can compile directly from SDL into your config structures.  
> 2. Specify the exact boolean/enum interface your `getToolsForScenario` needs for risk (e.g. `isHighRiskScenario`), and clearly state that it does **not** interpret raw `riskTags`. Propose a small helper signature that governance/infra agents will implement to supply this flag.  
> 3. Produce a concise implementation plan for wiring the ToolsPanel, tool events, and `handleToolEvent` into the existing `index.html` React mission view without breaking UX, including which components/functions will call `handleToolEvent` and how `scenarioId`, `nodeId`, and meta are threaded.  
>  
> **Pause/play:** Treat SDL updates from Agent F as upstream; if they are not yet implemented, work at the contract/spec level and mark runtime changes as TODOs.

#### 6.3 Agent I – Governance implementation hooks

> You are **Agent I**, focused on **Governance, Bias & Compliance Guardrails** for the TI Certification Simulation Platform.  
>  
> **Read these first, in order:**  
> - `TI-AGENT-ALIGNMENT.md` – governance & risk contracts and resolved dependencies with tools/SDL.  
> - `GOVERNANCE-BIAS-COMPLIANCE.md` – you maintain this spec.  
> - `TI-PLATFORM-PLAN.md` – Certification & Labels, Data & Observability, Workstream I.  
> - `ARCHITECTURE.md` – Event schema (v1), governance/fairness extensions, multi‑tenant readiness.  
>  
> **Tasks (in order):**  
> 1. Lock in MVP taxonomies and review SLAs: propose starter `riskTags`, `reasonCode`, and any bias checklist fields (small, clearly documented), plus minimal but clear SLAs for when scenarios/personas must be re‑reviewed.  
> 2. Define shared helper interfaces such as `getScenarioGovernance(scenarioId)` and `isHighRiskScenario(scenarioId)`, including how they expose flags to tools and analytics and how tenant‑level retention/export policies should be expressed for future Admin (D) and backend agents.  
> 3. Document a safe, free‑tier‑compatible pattern for automated Gemini‑based bias checks (offline, sampled jobs only) and where their outputs are recorded (governance docs vs `admin_action` events).  
>  
> **Pause/play:** Work at the spec/interface level only; when introducing new fields, mark them optional and additive and state how readers behave when fields are absent.

#### 6.4 Agent J – PlatformClient & runtime wiring

> You are **Agent J**, focused on **Performance, Scaling & Architecture Evolution** for the TI Certification Simulation Platform.  
>  
> **Read these first, in order:**  
> - `TI-AGENT-ALIGNMENT.md` – PlatformClient, event, tenancy/experiments/governance sections.  
> - `ARCHITECTURE.md` – Runtime Architecture, Event schema (v1), Workstreams J/E/K.  
> - `TI-PLATFORM-PLAN.md` – Technical Architecture, Admin & Multi‑Tenant Readiness, Observability, Workstream J.  
> - `README.md` – current app flow and what is implemented in `index.html`.  
>  
> **Tasks (in order):**  
> 1. Map all existing Firebase Auth, Firestore, and Gemini usage in `index.html` onto a thin `PlatformClient` wrapper, without adding new features. Identify which functions/components will call which methods.  
> 2. Specify, in light TypeScript/pseudocode, the exact request/response shapes for `startMission`, `submitDecision`, `getProfile`, `updateProfile`, `logEvent`, and `getMentorFeedback`, including optional `tenantId` and `experiments` fields and any logging/caching hooks required for performance and governance.  
> 3. Restate a concrete order of operations for gradually switching surfaces (events, profiles, missions, mentor feedback) from `firebase` to `api` mode while preserving UX.  
>  
> **Pause/play:** Assume no backend exists yet; design interfaces that can be implemented both in‑browser and on the server without changes. If you find contradictions with this alignment doc, call them out and suggest a reconciled contract instead of forking it.

---

### 7. Orchestration guidance for human operators

- Run **Agent F and Agent H in parallel** to stabilise SDL and tools contracts at the spec level.
- Then run **Agent I** to harden governance helpers, taxonomies, and SLAs on top of those contracts.
- Finally run **Agent J** so the PlatformClient and runtime wiring reflect the unified SDL, tools, and governance needs.
- After any agent introduces new cross‑cutting decisions or questions, update this alignment document first, then re‑run affected agents with the updated version referenced in their prompts.

