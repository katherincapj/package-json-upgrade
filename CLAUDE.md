# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Workspace layout

This directory is **not** itself a git repository — it's a folder containing several independent repos, each with its own `.git`. Always check which sub-repo you're in before running git or package-manager commands; there is no shared root config.

| Repo | What it is |
|---|---|
| `package-json-upgrade/` | A Claude Code multi-agent workflow (not an app) that upgrades dependencies across the other repos in this workspace. See below. |
| `mynyl-manage-access-frontend/` | Vue 2 + TypeScript frontend for the "Manage Access" flow (username/password/email/mobile self-service). |
| `mynyl-mobile-react/` | Expo / React Native mobile app. Currently near-default `create-expo-app` scaffold (file-based routing, minimal custom screens) — treat as early-stage. |
| `repo1/` | Empty placeholder, no content yet. |

---

## package-json-upgrade — dependency upgrade orchestrator

This repo has no application code. It defines a Claude Code agent system for safely bumping dependencies across multiple repos, driven entirely by `upgrade-policy.yaml`. The role specs live here as `.agent/*.agent.md` (source of truth for behavior); the **registered, invocable** versions — with YAML frontmatter and scoped tool access — live at the workspace root in `../.claude/agents/*.md` (`package-json-upgrade-orchestrator.md`, `planning-agent.md`, `upgrade-agent.md`). Claude Code only discovers `subagent_type`s from `.claude/agents/`, so if you change behavior, update the `.agent/*.agent.md` spec **and** port the change into the matching `.claude/agents/*.md` file — they can drift out of sync since they're two separate files.

- **`upgrade-policy.yaml`** is the single source of truth: max bump type (minor, never major), excluded/always-excluded packages, coupled package groups that must move together (e.g. `react`/`react-dom`/`@types/react`), retry-vs-skip rules by failure type, and validation order (`typecheck → lint → test → build`). Policy is never hardcoded into agent prompts — edit the YAML, not the agent files.
- **`.agent/package-json-upgrade-orchestrator.agent.md`** — coordinator. Loads policy, spawns one `planning-agent` per repo, presents Checkpoint 1 (plan approval) and Checkpoint 2 (failure review), enforces the `concurrency` cap when spawning `upgrade-agent`s, and produces the final report. Never touches package.json, lockfiles, or git itself.
- **`.agent/planning-agent.agent.md`** — read-only per-repo analyst. Detects the package manager from the lockfile present (`bun.lock(b)` > `pnpm-lock.yaml` > `yarn.lock` > `package-lock.json`), runs the outdated check, classifies packages against policy, validates every version against the npm registry, and returns a plan (upgrades/groups/excluded/userExcluded/userPinned). Never installs or modifies anything.
- **`.agent/upgrade-agent.agent.md`** — executes one repo's approved plan: installs, always commits `package.json` and the lockfile together, runs validation in policy order, classifies failures into a fixed taxonomy (`breaking-api`, `peer-conflict`, `tree-conflict`, etc.), and reverts to clean git state on failure. One commit per package/group — never batched. Requires a clean `git status` before starting.

When working in this repo: edit `upgrade-policy.yaml` to change behavior, not the agent prompt files. The three agent roles are strictly separated (orchestrator decides retry/skip, planner classifies, executor only reports facts) — don't blur those responsibilities when extending the system.

---

## mynyl-manage-access-frontend

Vue 2 / TypeScript / Vuex / Bootstrap-Vue app, built with `@vue/cli-service`.

**Commands** (run from inside `mynyl-manage-access-frontend/`):
```
npm run serve       # dev server with hot reload
npm run build        # production build to dist/
npm run test:unit    # Jest unit tests (tests/unit/**/*.spec.ts)
npm run test:e2e     # Cypress e2e tests (tests/e2e)
npm run lint         # eslint --fix via vue-cli-service
```
Run a single Jest test: `npx vue-cli-service test:unit tests/unit/example.spec.ts`.

Package manager note: the repo has both `package-lock.json` and a newer `pnpm-lock.yaml` + `pnpm-workspace.yaml` (with `allowBuilds` for native deps like `node-sass`, `bootstrap-vue`, `core-js`). The pnpm files are the actively maintained ones — prefer `pnpm install` unless told otherwise.

**Architecture**
- Feature-module layout under `src/app/<Feature>/`: each feature (currently `ManageAccess`) owns its own `router/`, `store/` (Vuex module + service), `service/`, `views/`, and `components/`. Routes are composed in `src/router/app-routes.ts` by importing each feature's router array and spreading it into the top-level route list; the Vuex store (`src/store/index.ts`) is set up for module registration the same way, with `vuex-persistedstate` persisting to `sessionStorage`.
- `src/app/shared/` holds cross-feature building blocks: `components/` (header, footer, nav, modals, breadcrumbs, cookie banner...), `services/` (`http-client`, `app-cookie`, `app-logger`, `app-storage`), and `form-validations/` (one rule file per field, e.g. `ruleEmail.ts`, `ruleSSN.ts`, `rulePassword.ts` — used with vee-validate).
- Path alias `@/*` → `src/*` (see `tsconfig.json`); `strict: true` but `noImplicitAny: false`.
- `src/environments/environment.ts` holds env-driven config (e.g. router `basePath`); actual secrets/URLs live in `.env` / `.env.production` (not committed content should be treated as sensitive — don't print them).
- **Deploy path**: this frontend is not deployed standalone. `checkin-springboot.bat` clones the separate `mynyl-manage-access` Spring Boot repo, copies this project's `dist/` build into `src/main/resources/static/public` there, and commits/pushes it as a static-resources update. Keep this coupling in mind when changing build output paths (`vue.config.js`) or the router `basePath`.

---

## mynyl-mobile-react

Expo (SDK ~54) / React Native 0.81 / React 19 app using **expo-router** file-based routing.

**Commands** (run from inside `mynyl-mobile-react/`):
```
npm install
npx expo start        # or: npm run android / npm run ios / npm run web
npm run lint           # expo lint
npm run reset-project  # moves scaffold code to app-example/, resets app/ to blank
```
No test script is currently defined.

**Architecture**: routes/screens live in `app/` (e.g. `app/(tabs)/index.tsx`, `app/_layout.tsx` for the root layout, `app/modal.tsx`); shared UI in `components/` (theming via `themed-text.tsx`/`themed-view.tsx` and `hooks/use-color-scheme*.ts`/`use-theme-color.ts`); path alias `@/*` → project root. The app is still close to the stock `create-expo-app` template — most screens are placeholder content, not yet NYL-specific.
