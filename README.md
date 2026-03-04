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
