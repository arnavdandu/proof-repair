# Where Were We? — VS Code Extension Design Plan (Pre-Build)

## 1) Problem Statement

Developers often pause work mid-stream and later return with low context:
- What files were touched?
- What decisions were made?
- What is finished vs in progress?
- What should happen next?

This extension captures coding-session activity and generates a concise LLM summary so you can re-enter a project quickly with the right context.

## 2) Product Vision

Build a **session memory layer for VS Code** that continuously records useful signals (edits, commands, diagnostics, git changes, optional notes) and transforms them into:
1. **Quick resume summaries** when reopening a workspace.
2. **Project re-acquaintance briefs** (structure, active areas, recent progress).
3. **Open-issues + next-steps lists** grounded in observed activity.

Core promise: _"In under 60 seconds, know where you left off and what to do next."_

## 3) Primary Users

- Solo developers juggling multiple repos.
- Team members context-switching across tasks.
- Anyone returning to a codebase after hours/days/weeks.

## 4) Core User Stories

1. **As a developer**, when I reopen VS Code, I see "Last session summary" with key changes and next actions.
2. **As a developer**, I can inspect a timeline of session events (edited files, commits, errors fixed, commands run).
3. **As a developer**, I can trigger "Summarize current work" at any time.
4. **As a developer**, I can copy/export summary into a PR, issue, or daily log.
5. **As a privacy-conscious user**, I can control what data is captured and whether LLM calls are local or remote.

## 5) Non-Goals (V1)

- Full autonomous project management.
- Rewriting code automatically based on logs.
- Capturing every keystroke or full file snapshots by default.
- Cross-IDE support (V1 is VS Code only).

## 6) High-Level Architecture

### 6.1 Components

1. **Event Collector (VS Code extension host)**
   - Subscribes to editor/workspace events.
   - Captures structured metadata (not raw keystrokes by default).

2. **Session Manager**
   - Starts/ends sessions (manual + idle-timeout heuristics).
   - Maintains active session state and session IDs.

3. **Local Event Store**
   - Persists events + derived snapshots in local SQLite/JSONL.
   - Supports retention and cleanup policies.

4. **Context Synthesizer**
   - Aggregates raw events into compact context packets:
     - touched modules,
     - changed files,
     - diagnostics trend,
     - git diff stats,
     - command history summary.

5. **LLM Summary Engine**
   - Prompt builder + provider abstraction (OpenAI-compatible/local).
   - Produces multiple summary formats (short/long/actionable).

6. **UI Layer**
   - Activity Bar view: recent sessions + summaries.
   - Commands: "Summarize Session", "Resume Context", "Export Summary".
   - Optional startup notification with quick resume card.

### 6.2 Data Flow

1. Session starts (workspace opened or command-triggered).
2. Event collector logs activity.
3. On trigger/interval/session end, synthesizer compacts context.
4. LLM engine generates summary + open issues + next steps.
5. Summary is stored and shown in UI; user can copy/export.

## 7) Event Model (V1)

Capture **high-signal, low-risk** events:

- File events: opened, edited, saved, renamed.
- Git events: branch changes, commit hash, changed files (from `git status`/`git diff --name-only`).
- Diagnostic events: counts by severity, top recurring errors.
- Task/terminal events (opt-in): executed command text + exit status.
- Navigation events: symbol/file switches (coarse-grained).
- User notes (manual): "checkpoint note" command.

### Example event schema

```json
{
  "ts": "2026-03-16T10:24:11Z",
  "sessionId": "sess_abc123",
  "type": "file.saved",
  "workspace": "/path/repo",
  "payload": {
    "path": "src/service/auth.ts",
    "language": "typescript",
    "locDelta": 24
  }
}
```

## 8) Summary Outputs

Generate three views from same context:

1. **TL;DR (30s)**
   - "You modified auth flow + tests, fixed 2 lint errors, 1 failing test remains."

2. **Progress Digest (2–3 min)**
   - What changed, why (inferred), where (files/modules), and confidence.

3. **Action List**
   - Open issues,
   - likely blockers,
   - concrete next commands/files.

## 9) Prompting Strategy

### 9.1 Prompt Inputs

- Session metadata (duration, active windows, branch).
- Top edited files + churn stats.
- Diagnostics before/after counts.
- Git status + latest commit message(s).
- Optional user notes/checkpoints.

### 9.2 Constraints

- Instruct model to separate **facts** (from logs) vs **inferences**.
- Force markdown sections:
  - `What changed`
  - `Current status`
  - `Open issues`
  - `Next 3 actions`
- Output must include confidence markers where uncertain.

### 9.3 Hallucination Control

- Provide only structured inputs; avoid full repository dumps.
- Add explicit instruction: "Do not invent files/errors not present in inputs."
- Optionally run post-check pass to verify file references exist.

## 10) VS Code UX Plan

### 10.1 Surfaces

- **Activity Bar: Where Were We?**
  - Session list with timestamps and branch.
  - Click to open stored summary.

- **Command Palette**
  - `Where Were We: Start Session`
  - `Where Were We: End Session`
  - `Where Were We: Summarize Last Session`
  - `Where Were We: Add Checkpoint Note`
  - `Where Were We: Export Summary`

- **Startup Resume Card**
  - Appears on workspace reopen.
  - Shows top 3 bullets + "Open full summary".

### 10.2 Interaction Details

- Auto-session starts when workspace becomes active.
- Idle > N minutes can split sessions.
- Manual checkpoint command inserts user-authored note into timeline.
- Summaries are cached; regenerate only when new events exist.

## 11) Privacy, Security, and Trust

- Default local storage under extension global state path.
- Opt-in controls for:
  - terminal command capture,
  - remote LLM usage,
  - including file snippets.
- Redaction options:
  - path anonymization,
  - secret-like token stripping,
  - command argument masking.
- "Local-only mode" supported via local model endpoint.

## 12) Performance Targets (V1)

- Event capture overhead: imperceptible (<5ms typical handler path).
- Summary generation kickoff: <1s for local context synthesis.
- Resume card availability on reopen: <2s when summary cached.

## 13) Suggested Tech Stack

- **Extension**: TypeScript + VS Code Extension API.
- **Storage**: SQLite (better queries) or JSONL (simpler V1).
- **LLM Provider Layer**: OpenAI-compatible adapter + local endpoint adapter.
- **Validation**: zod schemas for event + summary payloads.

## 14) Milestone Plan

### Milestone 0 — Instrumentation Spike (1 week)
- Capture file + diagnostic + git events reliably.
- Persist/reload sessions locally.

### Milestone 1 — Resume MVP (1–2 weeks)
- Session list UI + startup resume card.
- Rule-based summary (no LLM yet) as fallback baseline.

### Milestone 2 — LLM Summaries (1–2 weeks)
- Prompt builder + provider settings.
- TL;DR + digest + action list outputs.

### Milestone 3 — Hardening & Trust (1–2 weeks)
- Redaction/privacy settings.
- Better inference quality + confidence annotations.
- Export integrations (Markdown/clipboard).

## 15) Open Design Decisions

1. Session segmentation strategy:
   - Pure idle timeout vs explicit start/stop vs hybrid?
2. Storage format:
   - JSONL simplicity vs SQLite query power?
3. Capture depth:
   - Metadata-only default, optional snippets?
4. Summarization timing:
   - End-of-session only vs rolling updates?
5. Team workflows:
   - Single-user memory only, or optional shareable session briefs?

## 16) Success Metrics

- **Time-to-context**: median time from reopen to first productive edit.
- **Summary usefulness**: user rating after resume (1–5).
- **Adoption**: % sessions with summary viewed.
- **Actionability**: % summaries where "Next 3 actions" are used.

---

## Appendix A — Initial Command/Config Surface

- `whereWereWe.enable`
- `whereWereWe.captureTerminal` (default: false)
- `whereWereWe.captureFileSnippets` (default: false)
- `whereWereWe.model.provider`
- `whereWereWe.model.endpoint`
- `whereWereWe.privacy.redactPaths`
- `whereWereWe.session.idleMinutes`

## Appendix B — MVP Fallback Without LLM

If LLM unavailable, generate deterministic summary from stats:
- top edited files,
- diagnostics delta,
- uncommitted changes,
- last checkpoint note,
- suggested next file based on recency + errors.
