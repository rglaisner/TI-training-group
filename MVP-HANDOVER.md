# MVP V1 Handover

## Summary

MVP V1 delivers a **single-boundary mission flow** and **governance-aware tools** so the TI Certification Simulation Platform can be switched to an API backend later without changing callers, and so scenario governance and high-risk behaviour are observable and configurable.

### In scope for MVP V1

- **PlatformClient boundary (Workflow 1)**  
  All mission and profile operations go through `PlatformClient`.  
  - `startMission(request)` returns `{ sessionId, node, profile, labels, scenario, firstNodeId }` and optionally logs `mission_start`.  
  - `submitDecision(request)` performs state transition, persistence, and logging (`decision_made` / `mission_complete`) and returns `{ nextNode?, profile, labels, terminal }`.  
  - `startSim` and `handleDecision` in the React tree call these methods and sync UI state from the response. No direct Firebase/Gemini calls for mission flow outside `PlatformClient`.

- **Governance helpers (Workflow 2)**  
  - `getScenarioGovernance(appId, scenarioId)` reads Firestore `governance/{appId}/scenarios/{scenarioId}` and returns a governance doc or `null`.  
  - `isHighRiskScenario(governance)` returns true when `riskTags` includes any of `bias_sensitive`, `pii_adjacent`, `high_impact_label`.  
  - `seedScenarioGovernance(appId, scenarioId, overrides)` writes a stub governance document for testing (e.g. vt1).

- **Tools–governance wiring (Workflow 3)**  
  - `getToolsForScenario(scenario, options)` accepts `options.isHighRiskScenario`; when true, `evidence_board` is ensured in the tool list.  
  - The mission view resolves governance for the current scenario and passes `isHighRiskScenario` into `ToolsPanel`; high-risk scenarios show a short “Evidence board recommended” hint.

- **Event schema (Workflow 4)**  
  - All mission and tool events that should carry tenant/experiments/labels do so.  
  - `logEvent` supports optional top-level `tenantId`, `cohortIds`, `governanceContext`, `experiments`; a short comment documents that readers must treat missing values as “unknown”.

- **Scenario tools and externalData (Workflow 5)**  
  - Documented in `ARCHITECTURE.md` and in code: scenario may have `tools.enabledToolIds`; node may have `externalData.requests` for tools (e.g. Market & Comp Visualizer). SDL loaders may populate these later.

- **Governance admin view (Workflow 6)**  
  - In Admin Console, a “Scenario governance” block lists visible scenarios and shows governance status, release channel, risk tags, and last reviewed (read-only).  
  - “Refresh governance” loads governance from Firestore per scenario; “Seed governance for vt1” creates a stub approved doc for vt1.

---

## Smoke test checklist

1. **Sign in**  
   Use anonymous or test user.

2. **Start a mission**  
   From dashboard or mission log, start a mission (e.g. vt1).  
   - First node and choices render.  
   - No console errors.

3. **Make decisions and complete**  
   - Make choices; mentor feedback appears if the mentor persona module is enabled.  
   - Complete the mission to the summary screen.  
   - **Verify**: Profile has updated XP and labels.  
   - **Verify**: In Firestore, under `events/{appId}/users/{uid}/`, there are documents for `mission_start`, `decision_made`, and `mission_complete` with expected payload and optional fields (`tenantId`, `experiments`, `labelsBefore`/`labelsAfter` where applicable).

4. **Tool usage**  
   - Open a tool (e.g. Evidence Board), perform an action (e.g. save a note).  
   - **Verify**: A `tool_used` event is stored with `scenarioId`, `nodeId`, `toolId`, and context (`tenantId`, `experiments` if present).

5. **Governance (optional)**  
   - In Admin Console, open “Scenario governance”.  
   - Click “Refresh governance” and confirm scenario rows show status (or “No governance record”).  
   - Click “Seed governance for vt1”; refresh again and confirm vt1 shows “approved” and optional risk tags.  
   - Start vt1 and confirm the tools panel can show the “Evidence board recommended” hint when the scenario is high-risk (if you seeded risk tags).

---

## Known limitations

- **No backend API yet**  
  All logic runs in the browser; `PlatformClient` is implemented in “firebase” mode only. A future Node/TypeScript backend can implement the same contract and the frontend can switch to `mode: "api"` via configuration.

- **No automated bias scan**  
  Automated Gemini-based bias checks are out of scope for MVP; they are intended to be offline, sampled jobs (see `GOVERNANCE-BIAS-COMPLIANCE.md`).

- **No full Admin CRUD for governance**  
  Admins can view governance status and seed a stub for vt1; creating/editing governance documents (lifecycle, risk tags, review summary) is not implemented in the UI.

- **Auth**  
  Current auth is anonymous Firebase; enterprise auth (e.g. email, SSO) and tenant-bound identity are planned for later.

---

## Next steps (references)

- **TI-AGENT-ALIGNMENT.md** — Cross-workstream contracts and agent prompts (F, H, I, J).  
- **TI-PLATFORM-PLAN.md** — Epic checklist and workstreams (A–K).  
- **ARCHITECTURE.md** — Runtime architecture, event schema v1, scenario/node shape, PlatformClient.  
- **GOVERNANCE-BIAS-COMPLIANCE.md** — Scenario lifecycle, label governance, bias guardrails, retention/export policy direction.
