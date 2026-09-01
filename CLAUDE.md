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

## How policy is resolved

The orchestrator's Step 1 is the **only** place policy is resolved, and it runs parse → default →
validate. A key that is absent or empty gets a conservative default (`maxType` → `patch`,
`concurrency` → `1`, `requireApproval` → `true`, `validation.order` → all four steps, the retry
lists → empty). A key that is *present but invalid* — `maxType: banana`, `maxType: major`,
`concurrency: -1`, a label in both `retryOn` and `skipOn` — aborts the run instead. Missing means
"you didn't say"; wrong means intent that can't be guessed. A key that near-matches a real one
(`alwaysexclude`) also aborts, because that typo silently disables a safety list.

Everything defaulted or warned is printed in a `POLICY RESOLUTION` block before any agent spawns,
and repeated in the final report.

**Subagents never apply defaults.** `planning-agent` and `upgrade-agent` both open with an input
guard that refuses and names the missing key, because they can be invoked directly and bypass
Step 1 entirely. Adding a fallback there would create a second source of truth — resolution stays
in one place.

## Operating model

**Clone-and-point.** The session runs from the root of this repo; target repos are passed in as
absolute paths and added with `--add-dir`. This repo is the control plane — one policy file, one
allowlist, no configuration required in the target repos. `.claude/upgrade-policy.yaml` therefore
resolves against this repo, never against a repo being upgraded.

## Known drift to watch

**The upgrade agent never enters the target repo.** It runs `git status`, `npm install`, and
`git commit` assuming it is already there; the plan carries an absolute `repo` path that no step
tells it to use. Under clone-and-point the working directory is always this repo, so this blocks
every run. Same gap in the planner's outdated check.

**The `git:` policy block is dead config** — `defaultBranch`, `createIfMissing`, `baseBranch` are
referenced by no agent, so commits land on whatever branch is checked out.

`.claude/settings.local.json` is an allowlist grown from real runs (npm/pnpm outdated, npm view,
npm install, npm run, git add/commit, and several one-off `node -e` lockfile probes). Some entries
are hyper-specific to past sessions and can be generalized rather than appended to.
