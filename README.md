# package-json-upgrade

A Claude Code multi-agent system that performs safe minor/patch dependency upgrades across your
repos. It plans, installs, validates, and commits one package at a time — and reverts anything
that doesn't pass your test suite.

This repo contains **no application code**. There is no `package.json`, no build, no tests. It is
configuration: one policy file and three agent definitions.

---

## Before the first run — three things are broken

These are known gaps in the agent definitions. Fix them or the run will not work as described.

**1. The policy file path is wrong.** All three agent files tell the agent to read
`package-json-upgrade/upgrade-policy.yaml`. The file is actually at
`.claude/upgrade-policy.yaml`. The orchestrator hard-stops on step 1 with
*"upgrade-policy.yaml not found."*

**2. Nothing changes directory into the target repo.** The upgrade agent runs `git status`,
`npm install`, and `git commit` assuming it is already inside the repo being upgraded. The plan
carries an absolute `repo` path, but no step tells the agent to use it. Commands will run against
whatever directory the session started in.

**3. The `git:` policy block is never read.** `upgrade-policy.yaml` declares
`defaultBranch: feat/dependency-upgrades`, `createIfMissing`, and `baseBranch: main`. No agent
references any of it. Upgrades commit to **whatever branch is currently checked out** — which is
usually `main`. Check out an upgrade branch by hand before running, until this is wired up.

---

## Prerequisites

- Claude Code, and this repo cloned locally.
- The repos you want to upgrade, cloned locally, each with a clean `git status`. The upgrade
  agent refuses to start on a dirty working tree.
- Each target repo already installed (`node_modules` present) so validation scripts can run.
- Network access to the npm registry. Every version is confirmed against it before install.

---

## Running it

### 1. Start the session in this repo

```
cd package-json-upgrade
claude
```

This matters. Claude Code registers subagents from the `.claude/agents/` directory of the session's
working directory. Start anywhere else and the three agents do not exist. There is no user-level
or plugin copy.

### 2. Check out an upgrade branch in each target repo

Until gap 3 above is fixed, do this yourself:

```
cd ../your-repo
git checkout -b feat/dependency-upgrades
```

### 3. Ask for the upgrade

There is no slash command. Describe the job in plain language and name the repos:

```
Upgrade dependencies in these repos:
  C:/Users/you/repos/service-a
  C:/Users/you/repos/web-client
```

Add per-run overrides if you need them. These **merge with** the policy file — they never replace
it, so a package excluded in the YAML stays excluded even if you don't mention it:

```
exclude: [moment]
pin: [axios@1.4.0]
```

If a repo path doesn't exist, the run stops when all are invalid, and asks you whether to continue
when only some are.

---

## What happens during a run

The work moves through four stages.

**Stage 1 — Plan.** One read-only planning agent per repo, all launched at once. Each detects the
package manager from the lockfile (`bun.lock` > `pnpm-lock.yaml` > `yarn.lock` >
`package-lock.json`; on a tie the newest file wins), runs the outdated check, and sorts every
outdated package against the policy in a fixed order:

1. In `alwaysExclude` → excluded
2. In `exclude` → excluded
3. In your run's `exclude` → excluded
4. Bump larger than `maxType` → excluded
5. In a `coupled` group → upgraded as one unit or not at all
6. Everything left → a standalone upgrade

It then confirms each target version and three fallback versions against the npm registry. Anything
it cannot confirm is dropped, not guessed. Planning agents read only; they never install.

**Stage 2 — Checkpoint 1.** Review of the combined plan. **Currently skipped** —
`requireApproval` is `false`, so the run approves itself and continues. Set it to `true` to get a
say before anything is installed.

**Stage 3 — Upgrade.** Up to `concurrency` repos are processed at once (default 3). Within a repo,
each package or coupled group goes through the same loop:

```
install → git add package.json + lockfile → typecheck → lint → test → build → commit
```

Validation steps missing from the repo's `package.json` are skipped, not failed. Pass and the
change is committed on its own — **one commit per package or group, never batched**, and
`package.json` and the lockfile always land in the same commit. Fail and the agent labels the
failure, reverts to clean state, and moves to the next package. One bad package never stops the
rest of the repo.

**Stage 4 — Checkpoint 2 and report.** All failures are presented together at the end, never
mid-run. Also currently skipped while `requireApproval` is `false`. Failures the policy considers
retryable are offered as retries against the next fallback version; the rest are reported as
skipped with the reason. You get a per-repo summary of what upgraded, what failed, and what needs
a human.

---

## Configuring it

**Edit `.claude/upgrade-policy.yaml`. Do not edit the agent files to change behavior.** Policy is
deliberately kept out of the prompts so that what the system does is auditable in one place.

| Setting | Default | What it controls |
|---|---|---|
| `upgrade.maxType` | `minor` | Largest bump allowed. Majors are never attempted. |
| `upgrade.concurrency` | `3` | Repos upgraded simultaneously. |
| `upgrade.requireApproval` | `false` | Whether the two checkpoints pause for a human. |
| `exclude` | `[dayjs]` | Packages never touched, any repo. |
| `alwaysExclude` | react, vue, typescript, webpack, jest… | Framework and toolchain packages, never upgraded at all. |
| `coupled` | react trio, jest set, eslint set… | Packages that must move together. Authoritative — the agents do not infer coupling. |
| `validation.order` | typecheck, lint, test, build | Order gates run in. |
| `retry.retryOn` / `skipOn` | see file | Which failure types earn a fallback attempt. |
| `retry.maxFallbacks` | `3` | Lower versions tried before giving up. |
| `security.validateVersions` | `true` | Confirm every version on the registry first. Leave on. |

**Adding a package to `alwaysExclude` is the correct response to a package that keeps breaking
your build.** That is the intended way to teach the system.

### If a setting is missing or wrong

The orchestrator resolves the file once, in three stages — parse, default, validate — and reports
what it did before spawning anything.

**Missing or empty → a conservative default.** Absence means you didn't say, so the run takes the
narrowest reading and continues:

| Key | Default |
|---|---|
| `upgrade.maxType` | `patch` |
| `upgrade.concurrency` | `1` (serial) |
| `upgrade.requireApproval` | `true` (gate on) |
| `security.validateVersions` / `rejectUnknownVersions` | `true` |
| `lockfile.required` / `commitTogether` | `true` |
| `validation.order` | all four steps |
| `alwaysExclude` / `coupled` | the built-in lists |
| `exclude`, `retry.retryOn`, `retry.skipOn` | empty |
| `retry.maxFallbacks` | `3` |

An absent key never widens what a run may do.

**Present but invalid → the run stops.** A wrong value is intent that cannot be guessed:
`maxType: banana`, `maxType: major` (forbidden outright, so requesting it is a contradiction),
`concurrency: -1`, a non-boolean in a boolean field, an unknown step in `validation.order`, an
unknown label in a retry list, the same label in both retry lists, or a `coupled` group with one
package.

**A typo'd key also stops the run.** `alwaysexclude`, `always_exclude` and `excludes` are near-
matches of real keys, and silently ignoring one disables a safety list. Unrecognized keys that
resemble nothing are warned about and ignored.

**Missing or unparseable file → the run stops.** This is the loud case, unchanged.

Everything defaulted or warned appears in a `POLICY RESOLUTION` block before any work begins, and
again in the final report:

```
POLICY RESOLUTION
══════════════════════════════════════════
defaulted   upgrade.concurrency = 1
defaulted   validation.order = typecheck, lint, test, build
warning     unknown key 'notifications' — ignored
══════════════════════════════════════════
```

Note that `planning-agent` and `upgrade-agent` do **not** apply defaults. Invoked directly, they
refuse and name the missing key. Resolution happens in exactly one place.

`.claude/settings.local.json` is committed to this repo and holds the Bash permission allowlist
built up from previous runs. It has some very session-specific entries (one-off `node -e` lockfile
probes) that can be generalized. Note that `settings.local.json` is conventionally a personal,
untracked file — if the team is meant to share this allowlist, it belongs in `settings.json`.

---

## When something fails

The upgrade agent reports facts and never decides what to do next; the orchestrator applies the
retry policy. Every failure gets exactly one label:

| Label | Meaning | Policy default |
|---|---|---|
| `breaking-api` | Source calls an API the new version changed | Retry lower version |
| `behavior-change` | Compiles, but tests now fail | Retry lower version |
| `ts-incompatibility` | Type errors from the new version | Retry lower version |
| `lint-rule-change` | New or changed lint rules fire | Retry lower version |
| `tree-conflict` | Dependency tree could not resolve | Retry lower version |
| `stale-cache` | Package manager cache problem | Retry lower version |
| `peer-conflict` | Peer dependency requirements cannot be met | Skip — no lower version helps |
| `platform-env` | Node version, OS, or native build problem | Skip |
| `registry-auth` | Registry credentials rejected | Skip |
| `registry-network` | Registry unreachable after backoff | Skip |
| `pre-existing` | Already failing before any upgrade ran | Own report section, never retried |

A `pre-existing` result means your repo had a red test before the run started. Fix that first —
it makes every subsequent result ambiguous. It is deliberately absent from both `retryOn` and
`skipOn`: the failure predates the run and belongs to no package, so the orchestrator handles it
as its own outcome. Don't "fix" this by adding it to a retry list.

Any other label that appears in neither list is reported as **unrouted** — policy has no rule for
it, and the orchestrator will not improvise one. That's a gap in your `upgrade-policy.yaml` to
close.

---

## Guarantees and limits

**What it will not do.** Major version bumps. Batched commits. A commit of `package.json` without
its lockfile. An install of a version string it could not confirm on the registry. A start on a
dirty working tree. A change left behind after a failed validation.

**What it will not do that you might expect.** Push, open a pull request, resolve a merge conflict,
update code to match a changed API, or read a package's changelog. Every upgrade it produces still
needs review before it merges.

---

## Extending it

The three agents have deliberately separate jobs. Keep them separate.

- **`package-json-upgrade-orchestrator`** — reads policy once and hands the object to every
  subagent (subagents never read the YAML themselves), spawns agents, enforces concurrency, runs
  the checkpoints, and **owns the retry-vs-skip decision**. Touches no files, no package manager,
  no git.
- **`planning-agent`** — read-only. Classifies and validates. Makes no independent judgment calls;
  it applies the policy object it was handed.
- **`upgrade-agent`** — executes an approved plan. Installs, validates, commits, classifies
  failures. Reports what happened, never what should happen next.

Blurring these is the main way this system goes wrong. If an agent starts making a decision that
belongs to another, the run stops being auditable against the policy file.
