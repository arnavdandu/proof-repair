# Lean Proof Repair Tool — Design Plan (Pre-Build)

## 1) Problem Statement

Mathlib/dependency updates frequently invalidate previously working Lean proofs. Today, users manually inspect broken files, infer what changed in tactics/lemmas/typeclass behavior, and patch proofs with little guidance.

This project aims to provide a **focused proof-repair workflow** that:
- Anchors each failure to the **last-known-good line** from an older commit/version.
- Shows **old vs. new proof state side-by-side** for the same source location.
- Supports both **manual editing** and **LLM-assisted automated repair**.
- Leverages familiar **VS Code merge/diff resolution UX**.

## 2) Product Vision

Create a developer tool that turns proof breakage into a deterministic repair loop:
1. Detect breakages under new dependencies.
2. Recover corresponding successful context from an old snapshot.
3. Render old/new source and old/new proof states in side-by-side panes.
4. Let user resolve manually or request AI-generated candidate patch.
5. Validate by recompilation/tests and persist accepted fix.

## 3) Primary Users

- Lean project maintainers upgrading mathlib.
- Researchers with large formalization codebases.
- Contributors preparing compatibility PRs for dependency bumps.

## 4) Core User Stories

1. **As a maintainer**, I can select an old commit (or lockfile baseline) and current workspace, run analysis, and get a list of broken declarations.
2. **As a maintainer**, I can open a single broken declaration in VS Code with old/new code + old/new proof state aligned.
3. **As a maintainer**, I can request an LLM fix proposal constrained to this declaration and current imports.
4. **As a maintainer**, I can apply, inspect, and re-run Lean checking to confirm the fix.
5. **As a maintainer**, I can iterate candidate fixes and keep an audit trail of what changed and why.

## 5) Non-Goals (V1)

- Full automatic migration of an entire codebase with zero user review.
- Re-implementing Lean elaboration internals.
- Supporting all editors (V1 targets VS Code first).
- Solving semantic theorem changes requiring large refactors outside local declarations.

## 6) High-Level Architecture

### 6.1 Components

1. **Snapshot Manager**
   - Resolves baseline (old) vs target (new) revisions.
   - Checks out/builds each snapshot in isolated workdirs.

2. **Failure Indexer**
   - Runs Lean diagnostics on target snapshot.
   - Extracts broken declarations, locations, and messages.

3. **Context Extractor**
   - For each failure, maps to corresponding location in baseline.
   - Captures source slices and proof states at critical points.

4. **State Capture Engine**
   - Uses Lean/LSP info APIs to capture goals/context at cursor/line.
   - Produces normalized, machine-readable state records.

5. **Diff/Resolution UI Adapter (VS Code extension layer)**
   - Opens side-by-side view:
     - Left: baseline line + baseline proof state.
     - Right: target line + broken target proof state.
   - Adds actions: “Ask LLM”, “Apply Candidate”, “Re-check”.

6. **LLM Repair Engine**
   - Builds constrained prompt from local declaration, imports, diagnostics, states.
   - Produces minimal patch candidates.
   - Ranks candidates by compile success + heuristic quality.

7. **Validation Runner**
   - Rechecks declaration/file/project.
   - Returns pass/fail + remaining diagnostics.

8. **Change Ledger**
   - Records attempted fixes, prompts (optional redacted), model outputs, outcomes.

### 6.2 Data Flow

1. User selects baseline commit/tag + target branch.
2. Tool compiles both snapshots and collects metadata.
3. Tool identifies breakages in target.
4. For each breakage, tool aligns baseline declaration/line.
5. Tool captures old/new proof states at aligned point.
6. UI renders diff + states and offers repair actions.
7. User/manual or LLM applies patch.
8. Validator checks result; accepted fix committed to working tree.

## 7) Proof-State Alignment Strategy

This is the hardest part. Use layered matching:

1. **Declaration-level anchor (primary)**
   - Match by fully-qualified declaration name.
   - If renamed/moved, fallback to git similarity + syntax signature.

2. **Intra-declaration line mapping**
   - Compute token/AST-aware diff within declaration body.
   - Map broken line to nearest equivalent baseline span.

3. **State capture points**
   - At minimum: failing line cursor.
   - Optional: previous tactic boundary and declaration start.

4. **Confidence score**
   - Score mapping quality (exact name + high textual similarity + AST shape).
   - Warn user if confidence is low.

## 8) VS Code UX Plan

### 8.1 Main Views

- **Breakage List Panel**
  - Declaration name, file, error summary, mapping confidence, status.

- **Repair Workspace (side-by-side)**
  - Left editor: baseline declaration snippet.
  - Right editor: target editable declaration snippet.
  - Proof state panes below/alongside each editor.

- **Action Bar**
  - Ask LLM
  - Generate N candidates
  - Apply candidate
  - Re-check declaration
  - Re-check file

### 8.2 Interaction Details

- Selecting a failure opens a structured “resolve session”.
- Cursor-linked state refresh on both sides.
- Candidate patch preview in diff before apply.
- Inline diagnostics update after each check.
- Keyboard-centric flow for rapid triage.

## 9) LLM Integration Plan

### 9.1 Prompt Inputs (strictly scoped)

- Target declaration full text.
- Baseline declaration full text.
- Old/new proof states (at aligned location).
- Lean diagnostic message and code.
- Visible imports/local context.
- Project style constraints (optional).

### 9.2 Guardrails

- Never edit outside selected declaration by default.
- Minimize patch size.
- Preserve theorem statement unless explicitly allowed.
- Require compile check before surfacing as “viable”.

### 9.3 Candidate Loop

1. Generate candidate patch.
2. Apply in temp buffer.
3. Re-run Lean check.
4. Keep passing/partially-improving candidates.
5. Rank by: compile success > fewer sorrys > minimal diff.

## 10) Lean/Tooling Integration Details

- Prefer Lean LSP endpoints for diagnostics and goal state.
- Run two project environments:
  - Baseline (old lock/commit).
  - Target (current branch).
- Cache build artifacts and state snapshots for speed.
- Normalize file paths and module names across snapshots.

## 11) Robustness & Edge Cases

- Declaration deleted in new version.
- Statement changed (proof no longer type-correct by design).
- Massive refactors / file moves.
- Tactic scripts replaced by term proofs.
- New implicit args/typeclass priorities causing non-local failures.
- Timeouts in LLM or Lean checks.

Fallback behavior:
- Show best-effort mapping with low-confidence badge.
- Allow manual override of baseline anchor.
- Provide “open full file diff” escape hatch.

## 12) Performance Targets (V1)

- Initial indexing on medium project: < 2–5 minutes (cache cold).
- Open repair session for one failure: < 2 seconds after cached states.
- Re-check declaration candidate: < 1–3 seconds typical.

## 13) Security & Privacy

- Opt-in LLM usage.
- Redaction policy for local paths/secrets in prompts.
- Local-only mode (no remote model) via local inference endpoint.
- Store telemetry only with explicit consent.

## 14) Suggested Tech Stack

- **Core engine**: Python or Rust (or Node if tight VS Code integration priority).
- **Editor integration**: VS Code extension (TypeScript).
- **Process orchestration**: language-server subprocess + Lean CLI wrappers.
- **State/cache storage**: SQLite + file-based artifact cache.
- **Patch operations**: tree-sitter/Lean parser assisted, fallback textual patching.

## 15) Milestone Plan

### Milestone 0 — Discovery Spike (1–2 weeks)
- Validate extracting goal states at arbitrary cursor points.
- Validate baseline↔target declaration matching quality.
- Prototype side-by-side static view with synthetic data.

### Milestone 1 — Manual Repair Assistant (2–4 weeks)
- Failure index, mapping, VS Code side-by-side session.
- Manual edit + re-check loop.
- No LLM yet.

### Milestone 2 — LLM Candidate Assist (2–4 weeks)
- Prompt builder + candidate generation.
- Apply/preview/rerank based on compile checks.
- Session history and attempt ledger.

### Milestone 3 — Scale & Reliability (3–6 weeks)
- Caching, batching, better mapping heuristics.
- Better handling for refactors/renames.
- UX polish and keyboard workflows.

## 16) Open Design Decisions

1. Baseline selection model:
   - Explicit git ref only, or lockfile-based auto-detection?
2. Granularity of checks:
   - Declaration-level incremental checking vs file-level only?
3. Parser strategy:
   - Lean native AST APIs vs tree-sitter fallback.
4. Local model support:
   - Which API schema to standardize on (OpenAI-compatible, etc.)?
5. Multi-candidate UX:
   - Show top-3 automatically or one-at-a-time interactive?

## 17) Recommended Next Step (Before Coding)

Run a **technical spike** with one real repository upgrade scenario:
1. Pick a known commit pair that introduces breakages.
2. Implement minimal scripts to:
   - enumerate target errors,
   - map one declaration to baseline,
   - capture old/new proof states,
   - open a prototype side-by-side VS Code view.
3. Measure mapping quality and state-capture reliability.
4. Use results to lock V1 architecture choices.

---

If this plan looks right, the next deliverable should be a concrete V1 specification with:
- exact module boundaries,
- API contracts between engine and VS Code extension,
- and a task breakdown into implementable tickets.
