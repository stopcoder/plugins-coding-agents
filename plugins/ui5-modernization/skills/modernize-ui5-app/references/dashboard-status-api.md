# Dashboard Status API Reference

This document describes how the orchestrator pushes live progress updates to the UI5 Modernization Dashboard during a real modernization run.

## REST API

**Endpoint:** `POST http://localhost:3001/api/status`  
**Content-Type:** `application/json`  
**Input:** Any partial `ModernizationState` JSON object  
**Behavior:** Shallow-merges the input into `/tmp/ui5-modernization-state.json`  
**Response:** `{"ok": true, "status": "running"}`

### How to call from the orchestrator

Use `curl` in a Bash command. Example:

```bash
curl -s -X POST http://localhost:3001/api/status \
  -H "Content-Type: application/json" \
  -d '{"status":"running","currentPhase":1,"currentActivity":"Running initial UI5 linter analysis","overallProgress":0}'
```

To read current state: `GET http://localhost:3001/api/status`

### Shallow merge implications

`writeState()` does `{ ...existing, ...patch }`. This means:

- **Scalar fields** (strings, numbers, booleans, null) — replaced individually. Safe to send only changed fields.
- **`phases[]` array** — replaced entirely. You MUST send the full 5-element array with every call, even if only one phase changed. Omitting `phases` preserves the existing array; sending a partial array overwrites with the partial.
- **`deferredIssues[]` array** — same: replaced entirely if present, preserved if omitted.

## ModernizationState Schema

```typescript
interface ModernizationState {
  // Identity
  runId: string;                    // e.g. "run-1728900000000"
  status: "idle" | "running" | "completed";  // Controls dashboard screen

  // Preflight strip (top bar)
  project: string;                  // e.g. "sap.ui.demo.todo"
  projectPath: string;              // Filesystem path (not displayed)
  ui5Version: string;               // e.g. "OpenUI5 1.144.0"
  model: string;                    // e.g. "claude-opus-5-5" (not displayed)
  verificationMode: string;         // e.g. "Full autonomous"
  buildCommand: string;             // e.g. "npm run build" (not displayed)
  testCommand: string;              // e.g. "npm test" (not displayed)
  workingTreeStatus: string;        // "Clean" or "Dirty" — color-coded
  linterBaseline: number;           // Starting finding count, shown as "N findings"

  // Progress stats
  totalFindings: number;            // Same as baseline
  findingsFixed: number;            // Running count
  findingsRemaining: number;        // baseline - fixed (not displayed, derived)
  currentPhase: number;             // 1-based (0 = none)
  currentActivity: string;          // Free text shown in activity section
  elapsedSeconds: number;           // Rough estimate — dashboard interpolates client-side
  overallProgress: number;          // 0-100 integer: Math.round((findingsFixed / linterBaseline) * 100)

  // Phase details
  phases: PhaseState[];             // Exactly 5 elements — MUST send full array

  // Completion data
  deferredIssues: DeferredIssue[];  // Populated at completion
  reportPath: string;               // Path to MODERNIZATION-REPORT.md
  issuesPath: string;               // Path to MODERNIZATION-ISSUES.md
  startedAt: string | null;         // ISO 8601
  completedAt: string | null;       // ISO 8601
  phaseStartedAt: string | null;    // ISO 8601 — used by live timer
}

interface PhaseState {
  id: number;                       // 1-based
  name: string;                     // e.g. "Mechanical Baseline"
  description: string;              // e.g. "Autofix + Test Starter modernization"
  status: "pending" | "running" | "completed" | "failed";
  findingsBefore: number | null;    // Set when phase starts
  findingsAfter: number | null;     // Set when phase completes
  filesChanged: number | null;      // Set when phase completes
  commit: string | null;            // Short SHA, set on completion
  commitMessage: string | null;     // Full message, set on completion
  skills: string[];                 // Skills used in this phase
  durationSeconds: number | null;   // Elapsed seconds for this phase
}

interface DeferredIssue {
  file: string;                     // e.g. "webapp/Component.js"
  rule: string;                     // e.g. "no-deprecated-api"
  message: string;                  // Description
  reason: string;                   // Why it was deferred
}
```

## Status Lifecycle

```
"idle"  →  "running"  →  "completed"
```

- `"idle"` — dashboard shows start screen
- `"running"` — dashboard shows live progress (stepper, stats, activity, phase commits)
- `"completed"` — dashboard shows summary (stats, phase breakdown, deferred issues, report links)

The transition from `"running"` to `"completed"` triggers a 1.5s delay before switching screens.

## Initial Phases Array Template

Use this exact structure when sending the first `update-status` call after Phase 0:

```json
[
  {
    "id": 1,
    "name": "Mechanical Baseline",
    "description": "Autofix + Test Starter modernization",
    "status": "pending",
    "findingsBefore": null,
    "findingsAfter": null,
    "filesChanged": null,
    "commit": null,
    "commitMessage": null,
    "skills": ["modernize-test-starter"],
    "durationSeconds": null
  },
  {
    "id": 2,
    "name": "Foundation",
    "description": "manifest.json + Component.js modernization",
    "status": "pending",
    "findingsBefore": null,
    "findingsAfter": null,
    "filesChanged": null,
    "commit": null,
    "commitMessage": null,
    "skills": ["fix-manifest-json", "fix-component-async"],
    "durationSeconds": null
  },
  {
    "id": 3,
    "name": "Module System & Globals",
    "description": "JS globals, pseudo-modules, cycles, blind spots",
    "status": "pending",
    "findingsBefore": null,
    "findingsAfter": null,
    "filesChanged": null,
    "commit": null,
    "commitMessage": null,
    "skills": ["fix-js-globals", "fix-xml-globals", "fix-cyclic-deps", "fix-linter-blind-spots"],
    "durationSeconds": null
  },
  {
    "id": 4,
    "name": "Deprecated APIs",
    "description": "Replace deprecated theme and API usages",
    "status": "pending",
    "findingsBefore": null,
    "findingsAfter": null,
    "filesChanged": null,
    "commit": null,
    "commitMessage": null,
    "skills": ["fix-bootstrap-params"],
    "durationSeconds": null
  },
  {
    "id": 5,
    "name": "CSP Compliance",
    "description": "Externalize inline scripts",
    "status": "pending",
    "findingsBefore": null,
    "findingsAfter": null,
    "filesChanged": null,
    "commit": null,
    "commitMessage": null,
    "skills": ["fix-csp-compliance"],
    "durationSeconds": null
  }
]
```

## Update Frequency Guidelines

| Event | Push update? | Rationale |
|-------|-------------|-----------|
| Phase 0 complete (preflight ready) | **YES** | First dashboard population |
| Phase N starts | **YES** | Transitions stepper, shows findingsBefore |
| Sub-agent batch dispatched | **YES** | Activity text during long waits |
| Individual sub-agent returns | **NO** | Too granular; up to 8 concurrent |
| Regression check linter re-run | **NO** | Brief internal step |
| Phase N commit created | **YES** | Key data: commit, findings delta, files changed |
| Verification gate starts | **YES** | Long-running; user should see it |
| Verification gate completes | **YES** | Important pass/fail signal |
| Documentation commit done | **YES** | Final transition to "completed" |

**Total calls per run:** ~18–22 (roughly one every 15–20 seconds in a typical 6-minute run).

## Guard Clause

If the `curl` command to `http://localhost:3001/api/status` fails (connection refused, timeout, etc.), log a one-line note and continue. **Never retry failed status update calls. Never let a dashboard update failure interrupt the modernization workflow.** Dashboard reporting is best-effort.

## Dashboard Field Mapping

Which dashboard UI elements read which fields:

| UI Element | Fields Used |
|------------|-------------|
| Preflight strip | `project`, `ui5Version`, `verificationMode`, `workingTreeStatus`, `linterBaseline` |
| Stepper circles | `phases[].id`, `phases[].status`, `phases[].name` |
| Stepper durations | `phases[].durationSeconds` (completed phases only) |
| Phase info panel | `currentPhase`, `phases[].name`, `overallProgress` |
| Progress bar | `overallProgress` (0-100) |
| Status dot | `status`, active phase `.status` |
| Stats grid | `findingsFixed`, `totalFindings`, `elapsedSeconds`, `overallProgress`, count of `phases[].commit` |
| Activity text | `currentActivity` |
| Phase commits list | `phases[]` where `.status === "completed"` and `.commit` is truthy |
| Completed screen | `linterBaseline`, `elapsedSeconds`, `phases[]`, `deferredIssues[]`, `reportPath`, `issuesPath` |
