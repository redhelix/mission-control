# Hermes Mission Control Migration — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Fork mission-control into hermes-mission-control: rename the project, remove all OpenClaw code paths, and use Hermes Agent as the single backend (config, gateway, sessions, agents, skills).

**Architecture:** Staged migration. Phase 1: rename package and branding, remove OpenClaw-only modules and references. Phase 2: introduce Hermes-native config, agent source, gateway detection, and skills. Phase 3: task dispatch for Hermes and E2E/docs polish.

**Tech Stack:** Next.js 16, React 19, TypeScript 5, SQLite (better-sqlite3), existing Hermes integration (hermes-sessions.ts, /api/hermes). No OpenClaw CLI or openclaw.json.

**Design reference:** `docs/plans/2026-03-17-hermes-migration-design.md`

---

## Phase 1: Rename and Strip OpenClaw

### Task 1.1: Rename package and description

**Files:**
- Modify: `package.json`

**Step 1:** Update `package.json` name, description, and keywords.

Change:
- `"name": "mission-control"` → `"name": "hermes-mission-control"`
- `"description": "OpenClaw Mission Control ..."` → `"description": "Hermes Mission Control — dashboard for Hermes Agent orchestration"`
- In `keywords`: remove `openclaw`; add `hermes`, `hermes-agent` (keep agent, orchestration, dashboard, nextjs)

**Step 2:** Run typecheck.

Run: `pnpm typecheck`  
Expected: PASS (no changes to code yet)

**Step 3:** Commit.

```bash
git add package.json
git commit -m "chore: rename package to hermes-mission-control"
```

---

### Task 1.2: Update CLAUDE.md branding

**Files:**
- Modify: `CLAUDE.md`

**Step 1:** Replace first line and any OpenClaw/gateway references.

- Title: `# Mission Control` → `# Hermes Mission Control`
- Subtitle: mention Hermes Agent instead of OpenClaw where relevant
- Keep stack, setup, run, tests, directories, conventions, pitfalls; update only product name and gateway optional note to say "Hermes gateway" if applicable

**Step 2:** Commit.

```bash
git add CLAUDE.md
git commit -m "docs: CLAUDE.md branding to Hermes Mission Control"
```

---

### Task 1.3: Update README.md branding and quick start

**Files:**
- Modify: `README.md`

**Step 1:** Replace title, badges area, and "Why Mission Control?" with "Hermes Mission Control" and Hermes-focused value props. Remove OpenClaw from "Multi-gateway" bullet (Hermes only). Update clone URL to your fork if applicable (e.g. `hermes-mission-control`). Replace "OpenClaw fleet health check" with Hermes in install steps.

**Step 2:** Commit.

```bash
git add README.md
git commit -m "docs: README branding and quick start for Hermes Mission Control"
```

---

### Task 1.4: Add Hermes config and remove OpenClaw from config.ts

**Files:**
- Modify: `src/lib/config.ts`

**Step 1:** Add Hermes paths and remove OpenClaw-only exports.

- Add `hermesHome`: `process.env.HERMES_HOME || path.join(os.homedir(), '.hermes')`
- Add `hermesGatewayJsonPath`: `path.join(hermesHome, 'gateway.json')` (or from env if needed)
- Add `hermesBin`: `process.env.HERMES_BIN || 'hermes'`
- Remove (or stop exporting) `openclawHome`, `openclawStateDir`, `openclawConfigPath`, `openclawBin`, `clawdbotBin`, `gatewayHost`, `gatewayPort`, `logsDir`, `tempLogsDir`, `soulTemplatesDir` that are OpenClaw-specific. Keep `dataDir`, `dbPath`, `tokensPath`, `memoryDir`, `homeDir`, `gnap`, `retention` and any shared paths.
- For gateway URL: add or keep a single source (e.g. `hermesGatewayUrl` from env or derived from `hermesGatewayJsonPath`); remove OpenClaw gateway host/port from config.

**Step 2:** Find all imports of removed config keys and update in a follow-up task; for this task, ensure no remaining code in `src/lib/config.ts` references openclaw* and export the new hermes* keys.

**Step 3:** Run typecheck.

Run: `pnpm typecheck`  
Expected: FAIL (other files still reference old config). Note the list of files.

**Step 4:** Commit.

```bash
git add src/lib/config.ts
git commit -m "refactor: add Hermes config, remove OpenClaw keys from config.ts"
```

---

### Task 1.5: Remove openclaw-gateway and command.runOpenClaw usage

**Files:**
- Delete: `src/lib/openclaw-gateway.ts`
- Modify: `src/lib/command.ts` (remove `runOpenClaw` and any openclaw CLI calls)
- Modify: `src/lib/task-dispatch.ts` (remove `callOpenClawGateway`, `runOpenClaw`; stub or remove dispatch-to-gateway logic)

**Step 1:** Delete `src/lib/openclaw-gateway.ts`.

**Step 2:** In `src/lib/command.ts`, remove the function that runs the openclaw binary (e.g. `runOpenClaw`). If no other CLI is needed for Hermes in this phase, leave the file with only shared helpers or remove the file if empty.

**Step 3:** In `src/lib/task-dispatch.ts`, remove imports of `runOpenClaw` and `callOpenClawGateway`. Remove or stub the code path that dispatches tasks via OpenClaw gateway (e.g. leave a TODO or no-op for "dispatch to Hermes" in Phase 3).

**Step 4:** Run typecheck and fix any remaining references to `openclaw-gateway` or `runOpenClaw` across the repo.

Run: `pnpm typecheck`  
Fix all reported errors (grep for `openclaw-gateway`, `runOpenClaw`, `callOpenClawGateway`).

**Step 5:** Commit.

```bash
git add src/lib/openclaw-gateway.ts src/lib/command.ts src/lib/task-dispatch.ts
git commit -m "refactor: remove OpenClaw gateway and CLI; stub task dispatch"
```

---

### Task 1.6: Remove OpenClaw doctor

**Files:**
- Delete: `src/lib/openclaw-doctor.ts`
- Delete: `src/lib/openclaw-doctor-fix.ts`
- Modify: API route that serves doctor (e.g. `src/app/api/openclaw/doctor/route.ts` or similar): remove or return 410 Gone with message "OpenClaw doctor removed; use Hermes."

**Step 1:** Find doctor API route.

Run: `rg -l "doctor|openclaw-doctor" src/app`
Then remove or replace the handler.

**Step 2:** Delete doctor lib files and remove their imports everywhere.

**Step 3:** Run typecheck and tests.

Run: `pnpm typecheck && pnpm test -- --run`
Fix any failing tests that referenced doctor.

**Step 4:** Commit.

```bash
git add src/lib/openclaw-doctor.ts src/lib/openclaw-doctor-fix.ts src/app/api/...
git commit -m "chore: remove OpenClaw doctor"
```

---

### Task 1.7: Replace agent-sync with Hermes-backed agent source

**Files:**
- Modify: `src/lib/agent-sync.ts` (or replace with `src/lib/agent-hermes.ts`)

**Step 1:** Remove all logic that reads or writes `openclaw.json` and `agents.list`. Keep the sync result types if used by API. New behavior: either (a) return agents from MC DB only (no file sync), or (b) derive agents from Hermes (e.g. from hook-registered agents or a single "Hermes" agent). For Phase 1, minimal approach: sync endpoint returns agents from MC database only; no openclaw.json. Remove `getOpenClawConfigPath`, `readAgentsFromOpenClawConfig`, `syncAgentsFromOpenClawConfig`, `previewSyncDiff`, `writeAgentToOpenClawConfig`, etc.

**Step 2:** Update `src/app/api/agents/sync/route.ts` to use the new sync (DB-only or Hermes-derived). GET sync preview and POST sync should not reference openclaw.

**Step 3:** Run typecheck and tests.

Run: `pnpm typecheck && pnpm test -- --run`
Fix tests in `src/lib/__tests__/agent-sync.test.ts` (rewrite or remove openclaw.json-based cases).

**Step 4:** Commit.

```bash
git add src/lib/agent-sync.ts src/app/api/agents/sync/route.ts src/lib/__tests__/agent-sync.test.ts
git commit -m "refactor: agent sync Hermes-backed (no openclaw.json)"
```

---

### Task 1.8: Rename openclawId to hermesAgentId (or agentId) in schema and API

**Files:**
- Modify: `src/lib/validation.ts` (replace `openclaw_id`, `provision_openclaw_workspace`, `openclaw_workspace_path` with Hermes or generic equivalents or remove)
- Modify: `src/lib/db.ts` / schema if agent config stores openclawId
- Modify: `src/lib/coordinator-routing.ts`, `src/lib/mentions.ts`, `src/lib/agent-workspace.ts`, `src/lib/agent-sync.ts` (use hermesAgentId or agentId)
- Modify: `src/components/panels/settings-panel.tsx`, `src/components/panels/agent-detail-tabs.tsx`
- Modify: `src/app/api/agents/route.ts`, `src/app/api/agents/[id]/route.ts`

**Step 1:** In validation, remove or rename openclaw_id to hermes_agent_id (or keep a generic agent_id for routing). Update API request/response shapes.

**Step 2:** In DB, agents table config JSON: use `hermesAgentId` (or `agentId`) instead of `openclawId`. No DB migration required if config is JSON; just code that reads/writes the key.

**Step 3:** Grep and replace in codebase: `openclawId` → `hermesAgentId` (or `agentId`) in TypeScript/TSX; update UI labels "OpenClaw ID" → "Agent ID" or "Hermes agent ID".

**Step 4:** Run typecheck and tests.

Run: `pnpm typecheck && pnpm test -- --run`

**Step 5:** Commit.

```bash
git add src/lib/validation.ts src/lib/coordinator-routing.ts src/lib/mentions.ts src/lib/agent-workspace.ts src/components/panels/settings-panel.tsx src/components/panels/agent-detail-tabs.tsx src/app/api/agents/route.ts src/app/api/agents/[id]/route.ts
git commit -m "refactor: rename openclawId to hermesAgentId (or agentId) across API and UI"
```

---

### Task 1.9: Remove OpenClaw adapter and add Hermes adapter

**Files:**
- Delete: `src/lib/adapters/openclaw.ts`
- Create: `src/lib/adapters/hermes.ts` (implement FrameworkAdapter: register, heartbeat, reportTask, getAssignments, disconnect; framework = 'hermes')
- Modify: `src/lib/adapters/index.ts` (remove openclaw, add hermes)

**Step 1:** Create HermesAdapter that implements the same interface as OpenClawAdapter (eventBus for register/heartbeat/reportTask/disconnect; getAssignments from adapter.ts queryPendingAssignments).

**Step 2:** Register hermes in adapters index; remove openclaw. Update any code that called getAdapter('openclaw') to use 'hermes' or a config-driven default.

**Step 3:** Run typecheck.

Run: `pnpm typecheck`

**Step 4:** Commit.

```bash
git add src/lib/adapters/openclaw.ts src/lib/adapters/hermes.ts src/lib/adapters/index.ts
git commit -m "refactor: replace OpenClaw adapter with Hermes adapter"
```

---

### Task 1.10: Gateway runtime and URL for Hermes only

**Files:**
- Modify: `src/lib/gateway-runtime.ts` (read from Hermes config; remove openclaw.json registration)
- Modify: `src/lib/gateway-url.ts` (optional: add Hermes-specific default port/URL if documented by Hermes)
- Modify: `src/lib/websocket.ts` (client id: `openclaw-control-ui` → `hermes-mission-control` or value from env)
- Modify: `src/lib/device-identity.ts` if it references OpenClaw gateway

**Step 1:** In gateway-runtime, remove `registerMcAsDashboard` that wrote to openclaw.json. Replace with Hermes: e.g. ensure MC can register with Hermes gateway if Hermes supports allowed origins (or document manual step). Get gateway token/URL from Hermes env or gateway.json.

**Step 2:** In websocket.ts, set default client id to `hermes-mission-control` and update any error message that mentioned openclaw-control-ui.

**Step 3:** Run typecheck.

**Step 4:** Commit.

```bash
git add src/lib/gateway-runtime.ts src/lib/gateway-url.ts src/lib/websocket.ts
git commit -m "refactor: gateway runtime and WS client id for Hermes"
```

---

### Task 1.11: Skills registry — remove awesome-openclaw, add Hermes sources

**Files:**
- Modify: `src/lib/skill-registry.ts` (remove awesome-openclaw; keep or add agentskills.io / Hermes skills source)
- Modify: `src/lib/skill-sync.ts` (remove openclaw paths; use hermesHome/skills or env)

**Step 1:** In skill-registry.ts, remove RegistrySource 'awesome-openclaw' and searchAwesomeOpenclaw. Add a Hermes/agentskills.io source if available; otherwise keep only clawhub, skills-sh or generic.

**Step 2:** In skill-sync.ts, replace OPENCLAW_STATE_DIR / openclaw state paths with config.hermesHome or HERMES_HOME and paths like `~/.hermes/skills`, workspace skills under Hermes.

**Step 3:** Update API and UI that referenced awesome-openclaw (e.g. skills panel, registry search).

**Step 4:** Run typecheck and tests.

Run: `pnpm typecheck && pnpm test -- --run`
Fix tests in `tests/skills-registry.spec.ts` (remove or replace awesome-openclaw test).

**Step 5:** Commit.

```bash
git add src/lib/skill-registry.ts src/lib/skill-sync.ts tests/skills-registry.spec.ts
git commit -m "refactor: skills registry Hermes-only (remove awesome-openclaw)"
```

---

### Task 1.12: Security scan — OpenClaw category to Hermes

**Files:**
- Modify: `src/lib/security-scan.ts` (replace openclaw category with hermes: check ~/.hermes config files, permissions)

**Step 1:** Replace scanOpenClaw and openclaw category with scanHermesConfig: read ~/.hermes/config.yaml, gateway.json, .env if present; check file permissions and sensitive values. Remove references to openclaw.json.

**Step 2:** Update API and UI that display security categories (expect hermes instead of openclaw).

**Step 3:** Run typecheck and tests.

Run: `pnpm typecheck && pnpm test -- --run`
Update tests in `src/lib/__tests__/security-scan-fix-route.test.ts` and `tests/security-scan-api.spec.ts` (openclaw → hermes).

**Step 4:** Commit.

```bash
git add src/lib/security-scan.ts tests/security-scan-api.spec.ts src/lib/__tests__/
git commit -m "refactor: security scan Hermes config (remove OpenClaw)"
```

---

### Task 1.13: Scheduler and sessions — remove openclaw.json polling

**Files:**
- Modify: `src/lib/scheduler.ts` (remove "re-read openclaw.json" tick; keep Hermes session or agent refresh if any)
- Modify: `src/lib/sessions.ts` (use hermes state dir or state.db for session discovery; remove openclaw agents dir)

**Step 1:** In scheduler, remove the interval that re-read openclaw.json. Keep cron/task tick logic that does not depend on OpenClaw.

**Step 2:** In sessions.ts, ensure session discovery uses Hermes (hermes-sessions.ts + state.db) only; remove openclaw state dir / agents dir listing.

**Step 3:** Run typecheck.

**Step 4:** Commit.

```bash
git add src/lib/scheduler.ts src/lib/sessions.ts
git commit -m "refactor: scheduler and sessions Hermes-only"
```

---

### Task 1.14: Super-admin — remove or Hermes-ize tenants

**Files:**
- Modify: `src/lib/super-admin.ts` (remove openclaw_home, openclaw-gateway@.service, openclaw.json template; either remove tenant provisioning or add hermes_home / Hermes gateway install)
- Modify: `src/lib/migrations.ts` (tenants table: openclaw_home → hermes_home if we keep tenants)
- Modify: `src/lib/db.ts` (tenant type)

**Step 1:** In super-admin, remove all OpenClaw-specific steps (create-openclaw-state, seed-openclaw-template, openclaw-gateway@.service, openclaw-tenants env). Either (a) remove super-admin tenant provisioning for this fork, or (b) add equivalent Hermes steps (hermes_home, gateway install). Design doc says "optional super-admin for Hermes tenants" in Phase 3; for Phase 1, minimal: remove OpenClaw steps and stub or simplify tenant creation.

**Step 2:** In migrations and db, rename openclaw_home to hermes_home in tenants table if we keep it; add migration to rename column.

**Step 3:** Run typecheck and tests.

**Step 4:** Commit.

```bash
git add src/lib/super-admin.ts src/lib/migrations.ts src/lib/db.ts
git commit -m "refactor: super-admin Hermes-only (remove OpenClaw tenant steps)"
```

---

### Task 1.15: Config and runtime-env — hermesHome for .env path

**Files:**
- Modify: `src/lib/runtime-env.ts` (use config.hermesHome for .env path instead of openclawStateDir)
- Ensure `src/lib/config.ts` exports hermesHome and no openclaw* used elsewhere

**Step 1:** In runtime-env.ts, replace config.openclawStateDir with config.hermesHome (or equivalent) for resolving .env path.

**Step 2:** Grep for remaining openclawStateDir, openclawConfigPath, openclawHome in src; fix or remove.

**Step 3:** Run typecheck.

**Step 4:** Commit.

```bash
git add src/lib/runtime-env.ts src/lib/config.ts
git commit -m "refactor: runtime-env and config use hermesHome"
```

---

### Task 1.16: Store and UI — openclawUpdate and labels

**Files:**
- Modify: `src/store/index.ts` (rename openclawUpdate, openclawUpdateDismissedVersion to hermesUpdate, hermesUpdateDismissedVersion or remove if update check was OpenClaw-specific)
- Modify: All panels that show "OpenClaw" in labels or tooltips

**Step 1:** In store, remove or rename openclaw update state to hermes (or remove update check if it called OpenClaw releases). Update localStorage keys from mc-openclaw-update-dismissed to mc-hermes-update-dismissed if kept.

**Step 2:** Grep UI for "OpenClaw", "openclaw"; replace with "Hermes" / "Hermes Agent" or generic "Agent" where appropriate.

**Step 3:** Run typecheck and lint.

**Step 4:** Commit.

```bash
git add src/store/index.ts src/components/
git commit -m "refactor: store and UI labels Hermes Mission Control"
```

---

### Task 1.17: Auth guards and API routes — remove OpenClaw doctor route

**Files:**
- Modify: `tests/auth-guards.spec.ts` (remove /api/openclaw/doctor from protected list or replace with Hermes route)
- Remove or replace: `src/app/api/openclaw/` routes if any

**Step 1:** Remove OpenClaw API routes; ensure auth guards test does not reference removed routes.

**Step 2:** Run tests.

Run: `pnpm test:e2e -- tests/auth-guards.spec.ts` (or equivalent)

**Step 3:** Commit.

```bash
git add tests/auth-guards.spec.ts src/app/api/
git commit -m "chore: remove OpenClaw API routes from auth guards"
```

---

### Task 1.18: Tests and E2E — remove OpenClaw fixtures and configs

**Files:**
- Delete or rewrite: `tests/openclaw-harness.spec.ts`
- Modify: `tests/docker-mode.spec.ts` (remove OpenClaw gateway URLs; stub or use Hermes)
- Modify: `package.json` (remove test:e2e:openclaw* scripts)
- Delete: `scripts/e2e-openclaw/` if present
- Delete or repurpose: `playwright.openclaw.local.config.ts`, `playwright.openclaw.gateway.config.ts`

**Step 1:** Remove OpenClaw E2E scripts and configs. Keep generic E2E that does not depend on OpenClaw.

**Step 2:** Run E2E (generic).

Run: `pnpm test:e2e` (with default config)
Expected: Adjust or skip tests that assumed OpenClaw gateway until Hermes E2E is added.

**Step 3:** Commit.

```bash
git add package.json tests/ scripts/ playwright*.ts
git commit -m "chore: remove OpenClaw E2E and fixtures"
```

---

### Task 1.19: Remaining references and lint

**Files:**
- Grep entire repo for openclaw, OpenClaw, OPENCLAW_, clawdbot, CLAWDBOT_

**Step 1:** Run grep and fix remaining references (comments, docs, env examples). Leave only MISSION_CONTROL_* and Hermes-related env.

**Step 2:** Run full quality gate.

Run: `pnpm typecheck && pnpm lint && pnpm test -- --run`
Fix any remaining failures.

**Step 3:** Commit.

```bash
git add -A
git commit -m "chore: remove remaining OpenClaw references; lint clean"
```

---

## Phase 2: Hermes-Native Backend (Summary Tasks)

### Task 2.1: Gateway URL and health from Hermes config

**Files:** `src/lib/gateway-url.ts`, `src/lib/config.ts`, gateway health API

Read `~/.hermes/gateway.json` (or env HERMES_GATEWAY_URL) for WebSocket URL; implement gateway health check using Hermes gateway.pid or HTTP if Hermes exposes it. Remove any leftover OpenClaw port/host.

---

### Task 2.2: Agent list from Hermes hook / DB

**Files:** `src/app/api/agents/route.ts`, `src/lib/agent-sync.ts` (or agent-hermes.ts)

Ensure agents are created/updated from Hermes hook (existing /api/hermes hook) or from MC DB only; no file-based sync. Document how to register agents (hook vs manual).

---

### Task 2.3: Skills from Hermes and agentskills.io

**Files:** `src/lib/skill-registry.ts`, `src/lib/skill-sync.ts`

Wire registry to agentskills.io or Hermes skills path; sync from ~/.hermes/skills and workspace under Hermes.

---

## Phase 3: Task Dispatch and Polish (Summary Tasks)

### Task 3.1: Task dispatch for Hermes

**Files:** `src/lib/task-dispatch.ts`, Hermes hook or cron

Implement "assign task to Hermes" via: MC API for tasks + Hermes hook that polls or receives webhook, or document manual flow. No OpenClaw gateway call.

---

### Task 3.2: E2E and docs

**Files:** Playwright config, README, CLAUDE.md, deployment docs

Add Hermes E2E (optional: local Hermes + state.db); update all docs to Hermes Mission Control and Hermes Agent only.

---

## Execution Handoff

Plan complete and saved to `docs/plans/2026-03-17-hermes-migration-implementation.md`.

**Two execution options:**

1. **Subagent-Driven (this session)** — Dispatch a fresh subagent per task (or batch of tasks), review between batches, fast iteration.
2. **Parallel Session (separate)** — Open a new session with executing-plans skill, load this plan, and run task-by-task with checkpoints.

Which approach do you prefer?
