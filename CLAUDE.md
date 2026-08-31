# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

There is no application code, no `package.json`, and no build/test/lint commands. This repo is
**configuration only**: a Claude Code multi-agent system that performs safe minor/patch dependency
upgrades in *other* repos. Everything lives under `.claude/`:

```
.claude/upgrade-policy.yaml   # single source of truth for every upgrade decision
.claude/agents/               # the three registered subagents
.claude/settings.local.json   # accumulated Bash permission allowlist for upgrade runs
```

"Working in this repo" means editing policy or agent prompts — not writing code.

## The three agents

Claude Code discovers `subagent_type`s from `.claude/agents/*.md`, so these files are both the
spec and the live definition. Their responsibilities are strictly separated; do not blur them
when extending the system.

- **`package-json-upgrade-orchestrator`** (tools: Read, Agent, AskUserQuestion) — loads the policy
  once and passes the full policy object to every subagent (subagents never read the YAML
  themselves), spawns one planning agent per repo (all in parallel, uncapped), runs Checkpoint 1
  (plan approval) and Checkpoint 2 (failure review), enforces `upgrade.concurrency` when spawning
  upgrade agents, and owns the **retry-vs-skip decision** via `retry.retryOn` / `retry.skipOn`.
  Never touches files, package managers, or git.
- **`planning-agent`** (tools: Read, Bash, Glob, Grep, WebSearch) — read-only, one per repo.
  Detects the package manager by lockfile priority (`bun.lock(b)` > `pnpm-lock.yaml` > `yarn.lock`
  > `package-lock.json`; on ties, newest mtime wins and the ambiguity is noted), runs the outdated
  check, classifies packages against the policy in a fixed order, validates every version and its
  fallbacks against the npm registry **in one parallel batch**, and returns a plan object
  (`upgrades` / `groups` / `excluded` / `userExcluded` / `userPinned`). Always returns a plan, even
  an empty one. Never installs or modifies.
- **`upgrade-agent`** (tools: Read, Bash, Edit, Glob, Grep) — executes one approved plan. Requires
  a clean `git status` to start. Per package or group: install → `git add package.json <lockfile>`
  → validate in `validation.order` (skipping scripts absent from `plan.scripts`) → commit. One
  commit per package/group, never batched; `package.json` and the lockfile always in the same
  commit. On failure it classifies into a fixed taxonomy (`breaking-api`, `behavior-change`,
  `ts-incompatibility`, `lint-rule-change`, `tree-conflict`, `peer-conflict`, `platform-env`,
  `stale-cache`, `registry-network`, `registry-auth`, `pre-existing`), reverts to clean state, and
  **reports facts only** — it never decides retry vs skip or what version to try next.

Data flow: orchestrator reads policy → planning-agent returns a plan → Checkpoint 1 →
upgrade-agent returns a result → Checkpoint 2 → final report.

## Changing behavior

**Edit `.claude/upgrade-policy.yaml`, not the agent prompts.** Policy is never hardcoded into
prompts. The YAML controls: target git branch and its auto-creation, `maxType` (patch|minor —
never major), `concurrency`, `requireApproval` (whether the two checkpoints pause for the user;
currently `false`), registry version validation, lockfile co-commit, the `exclude` and
`alwaysExclude` lists, `coupled` package groups that must move together (authoritative — agents
do not infer coupling), retry/skip failure types, `retry.maxFallbacks`, and
`validation.order` (typecheck → lint → test → build).

Runtime overrides in the invoking prompt (`exclude:`, `pin:`) **merge with** policy — they never
replace it.

## Known drift to watch

The agent prompts refer to the policy file as `package-json-upgrade/upgrade-policy.yaml`; it
actually lives at `.claude/upgrade-policy.yaml`. If the orchestrator reports "upgrade-policy.yaml
not found", this path mismatch is the likely cause.

`.claude/settings.local.json` is an allowlist grown from real runs (npm/pnpm outdated, npm view,
npm install, npm run, git add/commit, and several one-off `node -e` lockfile probes). Some entries
are hyper-specific to past sessions and can be generalized rather than appended to.
