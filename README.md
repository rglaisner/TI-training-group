## TI Certification Simulation Platform

Prototype of a console-style simulation platform for Talent Intelligence, Workforce Planning, and People Analytics practitioners. Users play through realistic missions, interact with AI personas, and accumulate XP and labels of excellence that reflect sustained capability rather than a one-off exam.

The current implementation is a client-first React app embedded in `index.html`, backed by Firebase Auth/Firestore and Gemini for mentor feedback and optional TTS.

---

### Platform Data Flow

The diagram below shows how data moves through the system at runtime.

```mermaid
flowchart LR
  playerBrowser[PlayerBrowser] --> reactUI[ReactUI]
  reactUI --> firebaseAuth[FirebaseAuth]
  reactUI --> profileStore[FirestoreProfiles]
  reactUI --> simulationState[LocalSimulationState]
  reactUI -->|"LLM calls"| geminiAPI[GeminiAPI]
  reactUI -->|"events"| eventStore[FirestoreEvents]
```

- **PlayerBrowser**: the user’s browser running the TI Simulation Console (`index.html`).
- **ReactUI**: the React app that renders missions, decisions, rank, and the admin console.
- **FirebaseAuth**: anonymous authentication that assigns each player a stable UID.
- **FirestoreProfiles**: stores per-user XP, completed scenarios, and labels of excellence.
- **LocalSimulationState**: in-memory state for the current mission (current node, history, feedback).
- **GeminiAPI**: provides persona-conditioned mentor feedback and optional text-to-speech audio.
- **FirestoreEvents**: append-only stream of mission events (start, decisions, completion) used for analytics.

---

### Functional Simulation Flow

This diagram describes the functional steps when a user plays a mission.

```mermaid
flowchart LR
  selectScenario[SelectScenario] --> startMission[startSim]
  startMission --> firstNode[RenderFirstNode]
  firstNode --> makeDecision[MakeDecision]
  makeDecision -->|"optional"| mentorFeedback["PersonaFeedback (Gemini)"]
  makeDecision --> updateState[UpdateXP_Labels_State]
  updateState -->|"if not end"| nextNode[NextNode]
  updateState -->|"if end"| summary[ShowSummary]
```

1. **SelectScenario**: the user chooses a mission from the dashboard or mission log.
2. **startSim**: the app resets mission state, optionally plays TTS for the opening scene, and logs a `mission_start` event.
3. **RenderFirstNode**: the first narrative node and its choices are displayed.
4. **MakeDecision**: the user selects a choice; the app records the decision in local history.
5. **PersonaFeedback (Gemini)** (optional): if the mentor persona module is enabled, Gemini critiques the decision in the active persona’s voice.
6. **UpdateXP_Labels_State**: XP, completion flags, and labels of excellence are updated; profile and label data are persisted if the user is authenticated.
7. **NextNode vs ShowSummary**:
   - If the mission is not over, the next node is rendered (with optional TTS) and the loop continues.
   - If the mission ends, a summary screen is shown and a `mission_complete` event is logged.

For a deeper architectural breakdown, see `[ARCHITECTURE.md](ARCHITECTURE.md)` and the high-level master plan in `[TI-PLATFORM-PLAN.md](TI-PLATFORM-PLAN.md)`.

https://g.co/gemini/share/0b15761aa564

Talent Intelligence Professional Console

1. Project Overview

The Talent Intelligence (TI) Professional Console is an immersive, AI-driven training environment designed to calibrate the decision-making quality of Talent Intelligence practitioners. It transforms theoretical TI frameworks into a high-stakes "Mission Control" simulation where every choice has a quantified business impact.

2. Historical Context

The Origins: The .jsx Era

The project originated as a static React component (ti-training-group-v11.jsx) authored by Toby Culshaw.

Original Core: It served as a robust decision-tree engine containing 13 distinct scenarios ranging from foundational definitions to executive AI strategy.
The Transition: The current iteration has refactored this foundational logic into a single-file, mobile-responsive HTML application. While the original scenario data remains the "brain" of the experience, the interface has shifted from a vertical list-based UI to a non-linear SVG Career Pathway.
3. Data Flow & Architecture

The console operates on a three-tier architecture:

A. Intelligence Layer (Gemini 2.5)

Strategic Evaluation: Every decision is sent to the gemini-2.5-flash-preview-09-2025 model. The AI doesn't just check if a choice is "right"; it evaluates the strategic nuance and returns a score (0-100).
Multimodal Feedback: gemini-2.5-flash-preview-tts provides real-time voice-overs for stakeholders, increasing psychological immersion.
Performance Memos: Post-mission, the AI generates a business impact summary based on the user's specific performance history.
B. Persistence Layer (Firebase Firestore)

Identity Gating: * Professional Accounts: Authenticated via Google. Data is persisted in /artifacts/{appId}/users/{userId}/profile.
Guest Access: Authenticated anonymously. Achievement data is volatile and session-bound.
Career Progress: Firestore tracks xp, completedScenarios, and the highPerformanceCount used for fast-tracking.
C. UI Layer (React & Tailwind)

Responsive Engine: A mobile-first design that adapts from a sidebar-based desktop console to a bottom-nav mobile interface.
Visual Pathway: A dynamic SVG tree that gates access to scenarios based on user maturity and performance scores.
4. User Guide: Console Operations

Initialization & Security Clearance

Identity Selection: Choose "Professional Account" to enable cross-session saving. Use "Guest Mode" for one-time sessions.
The Briefing: New users are presented with an Intelligence Briefing (Onboarding Tour). Do not skip this if you are unfamiliar with the HUD (Heads-Up Display).
Navigating the Pathway

Locked Nodes: Scenarios appear grayed out until the preceding node is completed.
Current Objective: The next available mission pulses blue.
Career Fast-Tracking: If the AI Mentor detects elite-level strategic thinking (Scores > 85 on two consecutive missions), the console will automatically unlock advanced levels, bypassing foundation requirements.
Field Interaction

Scenario Feed: Read the stakeholder's prompt in the main feed.
Audio Triggers: Ensure volume is on; stakeholders will speak their prompts.
Decision Matrix: Choices are pinned to the bottom of the screen. Tap a choice to execute.
Mentor Feedback: After a choice, the AI Mentor will provide a real-time critique. This feedback is critical for increasing your Intelligence Score.
5. Technical Maintenance

API Key: As Secret in Github repo (LLM); or hard coded (firebase)
Level Adjustments: New scenarios can be added to the SCENARIOS array; the SVG Pathway will automatically scale to accommodate new nodes.
Firebase Rules: Ensure Firestore rules allow read/write access to the artifacts/{appId}/users/{userId}/ path for authenticated users.
Developed for the Talent Intelligence Collective. Refactored from original works by Toby Culshaw.
