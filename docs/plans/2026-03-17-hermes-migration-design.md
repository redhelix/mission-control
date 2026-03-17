# Hermes Mission Control — Design & Migration Plan

**Date:** 2026-03-17  
**Scope:** Fork mission-control into a Hermes-only dashboard; rename to **hermes-mission-control**.

---

## 1. Overview

Mission Control is currently tightly coupled to OpenClaw (config, gateway CLI/WS, agent sync from `openclaw.json`, doctor, skills registry, super-admin). This design describes a **Hermes-only** fork: remove all OpenClaw code paths, rename the project to **hermes-mission-control**, and use Hermes Agent as the single backend (config: `~/.hermes`, gateway: Hermes gateway, sessions: `state.db`, skills: Hermes/agentskills.io).

**Approach:** Staged migration — Phase 1 fork + rename + strip OpenClaw; Phase 2 Hermes-native config/agents/gateway; Phase 3 task dispatch and polish.

---

## 2. Product Identity

| Item | From | To |
|------|------|-----|
| Package name | `mission-control` | `hermes-mission-control` |
| Repo / product name | Mission Control (OpenClaw) | Hermes Mission Control |
| Description | OpenClaw Mission Control — agent orchestration dashboard | Hermes Mission Control — dashboard for Hermes Agent orchestration |
| UI branding | Mission Control, OpenClaw references | Hermes Mission Control, Hermes Agent |
| Keywords | openclaw, agent, orchestration, ... | hermes, hermes-agent, agent, orchestration, ... |

**Files to update for rename/branding:**  
`package.json`, `README.md`, `CLAUDE.md`, `CHANGELOG.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`, `SKILL.md`, any `docs/` and `wiki/` references, installer scripts and docs that mention "Mission Control" or "OpenClaw" in a product sense.  
**Env vars:** Keep `MISSION_CONTROL_*` for MC’s own data (DB, tokens, build) unless we explicitly rename to `HERMES_MISSION_CONTROL_*` (optional; design keeps `MISSION_CONTROL_*` for MC app state).

---

## 3. Backend: Hermes Only

- **Config root:** `~/.hermes` (or `HERMES_HOME` if we support it). No `~/.openclaw` or `openclaw.json`.
- **Hermes config files:** `~/.hermes/config.yaml`, `~/.hermes/gateway.json`, `~/.hermes/.env` — read-only where needed for gateway URL, auth, hooks.
- **MC data:** Unchanged — `MISSION_CONTROL_DATA_DIR` (default `.data/`), `MISSION_CONTROL_DB_PATH`, etc. MC’s SQLite and tokens remain separate from Hermes state.

---

## 4. Components to Remove (OpenClaw-Only)

- **Config / paths:** All `openclawStateDir`, `openclawConfigPath`, `openclawHome`, `openclawBin`, `clawdbotBin` from `src/lib/config.ts` and any code that reads `openclaw.json` or `~/.openclaw`.
- **Gateway (OpenClaw):** `src/lib/openclaw-gateway.ts` (`callOpenClawGateway`, `parseGatewayJsonOutput`), `runOpenClaw` in `src/lib/command.ts`. All `openclaw gateway call` and OpenClaw gateway WS client id (`openclaw-control-ui`).
- **Agent sync from OpenClaw:** `src/lib/agent-sync.ts` logic that reads/writes `openclaw.json` `agents.list`. Replace with Hermes-backed agent source (see below).
- **Doctor:** `src/lib/openclaw-doctor.ts`, `src/lib/openclaw-doctor-fix.ts`, and API routes that invoke or parse `openclaw doctor`.
- **Skills (OpenClaw-specific):** `awesome-openclaw` registry in `src/lib/skill-registry.ts`; OpenClaw-specific skill paths in `src/lib/skill-sync.ts`. Keep or generalize to agentskills.io / Hermes skills.
- **Super-admin (OpenClaw tenants):** Tenant fields and scripts that reference `openclaw_home`, `openclaw-gateway@.service`, `openclaw.json` template, `OPENCLAW_*` in generated env. Either remove super-admin or redefine for Hermes (per-tenant `~/.hermes` or Hermes gateway install).
- **Adapter:** Remove `OpenClawAdapter` and `openclaw` from adapter registry; add `HermesAdapter` if we need framework events from Hermes (e.g. hook-driven).
- **Validation / API:** Remove `openclaw_id`, `provision_openclaw_workspace`, `openclaw_workspace_path` from validation schemas and API where they are OpenClaw-specific; replace with Hermes agent identity where needed.
- **Tests / E2E:** Remove or replace OpenClaw-specific tests and fixtures: `tests/openclaw-harness.spec.ts`, `tests/docker-mode.spec.ts` (OpenClaw gateway URLs), `scripts/e2e-openclaw/`, `playwright.openclaw.*.config.ts`, `test:e2e:openclaw*` scripts. Add Hermes-based E2E where applicable.
- **Docs / scripts:** Remove or rewrite `openclaw_hardening_guide.md`, OpenClaw-specific install steps, and any references to OpenClaw gateway/CLI in install and deployment docs.

---

## 5. Components to Replace or Add (Hermes-Native)

- **Config module:** New or refactored `src/lib/config.ts` (or hermes-config): `hermesHome` (e.g. `process.env.HERMES_HOME || join(homedir(), '.hermes')`), `hermesGatewayJsonPath`, optional `hermesBin`. Remove openclaw* keys.
- **Gateway connection:** Use Hermes gateway only. If Hermes exposes a WebSocket or HTTP API for dashboards, use it for live status/sessions. Otherwise: gateway “running” via `~/.hermes/gateway.pid`, session list from `state.db` (existing `hermes-sessions.ts`). Websocket client id: e.g. `hermes-mission-control` (and document if Hermes gateway requires a specific client id).
- **Agent list:** Source of truth is no longer `openclaw.json` `agents.list`. Options: (1) MC-owned `agents` table populated by Hermes hook (e.g. from `/api/hermes` hook install) and/or (2) derive from Hermes sessions / config. No sync from openclaw.json; add Hermes-specific sync or hook-driven registration.
- **Agent identity:** Replace `openclawId` with a single canonical id: `hermesAgentId` (or keep a generic `agentId` in config). Update DB schema, API, and UI (settings panel, agent detail, coordinator routing, mentions) to use the new field name and stop reading/writing `openclawId`.
- **Task dispatch:** OpenClaw used `callOpenClawGateway` to push tasks. Hermes has no equivalent CLI. Redesign: e.g. (1) MC creates tasks and exposes them via API; a Hermes hook or cron job polls MC and invokes Hermes agent, or (2) webhook from MC to a local Hermes endpoint if Hermes adds one. Document “task assignment” flow for Hermes in the design/impl plan.
- **Skills:** Point skill registry and sync at Hermes skills (e.g. `~/.hermes/skills`, agentskills.io). Remove awesome-openclaw; add or keep a Hermes-compatible registry source.
- **Security scan:** Replace “openclaw” category and openclaw.json checks with Hermes config checks (e.g. `~/.hermes/config.yaml`, `gateway.json`, permissions).
- **Sessions:** Already using `hermes-sessions.ts` and `state.db`. Keep and ensure all session list/detail flows use Hermes sessions only; remove OpenClaw session paths if any.
- **Gateway runtime / URL:** `gateway-url.ts` and gateway health: derive URL from Hermes gateway config (e.g. `gateway.json`) or env (e.g. `HERMES_GATEWAY_URL`); remove OpenClaw host/port/token logic from `gateway-runtime.ts` and related.

---

## 6. Data & Schema

- **DB:** Keep existing MC tables (tasks, agents, sessions, etc.). No need to rename DB file; can keep `mission-control.db` or introduce `hermes-mission-control.db` for clarity.
- **Agents table:** Add or rename column for Hermes agent id (e.g. `hermes_agent_id` or repurpose `config` JSON to hold `hermesAgentId`). Stop writing `openclawId` in config JSON.
- **Tenants (if super-admin kept):** Replace `openclaw_home` with e.g. `hermes_home` and adjust provisioning to Hermes (gateway install, config template).
- **Migrations:** New migration(s) to add Hermes-specific columns or backfill agent identity from existing config; optionally remove OpenClaw-only columns if any.

---

## 7. Environment Variables

- **Remove / deprecate:** `OPENCLAW_*`, `CLAWDBOT_*`, `MISSION_CONTROL_OPENCLAW_*` from code and docs. Do not read them for backend behavior.
- **Keep:** `MISSION_CONTROL_DATA_DIR`, `MISSION_CONTROL_DB_PATH`, `MISSION_CONTROL_BUILD_*`, `AUTH_*`, `API_KEY`, etc.
- **Add / document:** `HERMES_HOME` (optional, default `~/.hermes`), `HERMES_BIN` (optional), `HERMES_GATEWAY_URL` or equivalent if needed for dashboard connection. `NEXT_PUBLIC_GATEWAY_OPTIONAL` remains; clarify it applies to Hermes gateway.

---

## 8. UI & Copy

- Replace “OpenClaw” and “openclaw” in user-facing strings with “Hermes” / “Hermes Agent” where appropriate.
- Settings, agent detail, coordinator routing: “OpenClaw ID” → “Agent ID” or “Hermes agent ID”.
- Connection status and gateway health: label as Hermes gateway, not OpenClaw.
- README, CLAUDE.md, and in-app help: describe Hermes Mission Control as the dashboard for Hermes Agent; remove OpenClaw quick-start and references.

---

## 9. Testing Strategy

- **Unit tests:** Remove or rewrite tests that depend on OpenClaw (openclaw-gateway, openclaw-doctor, agent-sync from openclaw.json, security-scan openclaw category). Add tests for Hermes config, session scan, and gateway detection.
- **E2E:** Drop OpenClaw-specific Playwright configs and scripts. Add Hermes E2E (e.g. local Hermes install + gateway.pid + state.db) where feasible; otherwise stub gateway/sessions for CI.
- **Lint/typecheck:** Update any references to removed modules so lint and typecheck pass.

---

## 10. Phases Summary

| Phase | Focus | Deliverables |
|-------|--------|--------------|
| **1** | Fork + rename + strip OpenClaw | New repo/package `hermes-mission-control`; branding and docs updated; all OpenClaw-only code and tests removed; config and gateway use Hermes only (or stubs). |
| **2** | Hermes-native backend | Config from `~/.hermes`; agent list from Hermes/hooks; gateway status from Hermes; skills from Hermes/agentskills.io; security scan for Hermes config. |
| **3** | Task dispatch & polish | Task assignment flow for Hermes (hooks/polling/API); optional super-admin for Hermes tenants; E2E and docs updated. |

---

## 11. Out of Scope (This Design)

- Supporting both OpenClaw and Hermes in the same codebase.
- Changes to Hermes Agent itself (gateway protocol, hooks, or config format).
- Migrating existing OpenClaw users’ data (this is a clean fork; no automatic migration from openclaw.json).

---

## 12. Implementation Plan

The next step is to produce a **detailed implementation plan** (file-level tasks, order of operations, verification steps) using the writing-plans skill, so the fork and migration can be executed in batches with review checkpoints.
