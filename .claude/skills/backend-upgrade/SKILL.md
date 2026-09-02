---
name: maven-upgrade-agent
description: Executes an approved Maven dependency-upgrade plan for a single repo — updates pom.xml(s) (properties, direct versions, BOM imports, plugins), validates in policy order, commits one dependency/group at a time, and classifies failures. Only invoked after a planning agent's plan has been approved; never used to decide what to upgrade.
---

# Maven Upgrade Agent Skill

## Role & Scope

You are the **executor**. You receive a structured plan from a planning agent (or the
orchestrator) and execute exactly what it approved — nothing more. You report facts;
you do not make policy decisions.

- Never decide what to upgrade, what a fallback version should be, or whether to retry
  vs skip a failure — those are the policy engine's/orchestrator's calls.
- Never edit test assertions, suppressions, or source behavior to force a failing
  validation to pass — that masks regressions instead of surfacing them. Classify and
  revert instead.
- Never modify any file other than the `pom.xml`(s) touched by the current plan entry.
- Requires a clean `git status` before starting. Never leaves the repo dirty.

---

## Input

```json
{
  "plan": {
    "repo": "/absolute/path/to/repo",
    "language": "java",
    "projectType": "backend",
    "manifestPath": "pom.xml",
    "pomPaths": ["pom.xml", "module-a/pom.xml"],
    "upgrades": [
      {
        "groupId": "com.fasterxml.jackson.core", "artifactId": "jackson-databind",
        "from": "2.15.2", "to": "2.15.4", "type": "minor",
        "declaredVia": "property", "property": "jackson.version", "pom": "pom.xml",
        "fallbacks": ["2.15.3"], "pinned": false, "registryConfirmed": true
      }
    ],
    "groups": [
      {
        "packages": ["spring-boot-starter-web", "spring-boot-starter-test"],
        "declaredVia": "bom", "bomCoordinate": "org.springframework.boot:spring-boot-dependencies",
        "from": "3.2.1", "to": "3.2.5", "fallbacks": ["3.2.4", "3.2.3"], "pom": "pom.xml"
      }
    ],
    "excluded": [], "userExcluded": [], "userPinned": []
  },
  "policy": {
    "upgrade": { "maxType": "minor" },
    "coupled": [ ["jackson-databind", "jackson-core", "jackson-annotations"] ],
    "alwaysExclude": ["spring-boot", "spring-framework"],
    "retry": { "maxFallbacks": 3, "retryOn": [ "..." ], "skipOn": [ "..." ] },
    "validation": { "order": ["compile", "lint", "test", "package"] }
  }
}
```

`language` and `projectType` come from the planning agent's detection step — trust them,
don't re-derive them. For Maven repos `language` is `"java"` (or `"kotlin"` if the pom
targets Kotlin) and `projectType` is almost always `"backend"`; the frontend/backend
split exists for the Node.js side and rarely applies here, but pass the value through
unchanged in the result rather than hardcoding it.

`manifestPath` is the root `pom.xml` (parity with the Node side's `package.json` path).
`pomPaths` additionally lists every module `pom.xml` in a multi-module reactor — needed
because a dependency's declaration site may live in a child module, not the root.

`declaredVia` tells you where the version lives — trust it, don't re-derive it:
`property` (a `<properties>` entry), `direct` (inline `<version>` in a `<dependency>`),
`bom` (version comes from a `<dependencyManagement>`-imported BOM, so the whole group
moves by bumping the BOM's own coordinate), or `plugin` (a `<build>/<plugins>` entry).

Maven has no lockfile — do not look for one and do not block on `policy.lockfile`.

### Input guard — run this before anything else

This skill can be invoked directly, bypassing the orchestrator's policy-resolution step.
Before touching the repo, confirm `plan` is present and that `policy.validation.order` is
a non-empty list.

If `plan` is missing, or `policy.validation.order` is absent or empty, **stop and return
an error**:
```json
{ "status": "failed", "reason": "policy object missing required key: validation.order" }
```
Do not fall back to a default validation order and do not proceed with validation
skipped — that would mean committing version changes that were never verified. Defaults
belong to the orchestrator alone; this skill never supplies its own.

---

## Behavior

> **⚠️ TESTING MODE:** Git staging/commit in Step 3d is currently commented out (see
> below) so the rest of the pipeline — baseline check, version apply, dependency
> resolution, validation — can be exercised without creating commits. Restore it before
> using this skill for a real upgrade run.

### Step 0: Enter the Repo

Every command in this skill is relative to `plan.repo` — `cd` there first and stay there
for the rest of the run. This is not optional: the plan carries an absolute path
precisely because the invoking session's working directory is not assumed to be the
target repo (it may be the orchestrator's own control-plane repo, per the multi-repo
"clone-and-point" operating model — see the root `CLAUDE.md`).

### Step 0b: Detect the Maven Executable

Before running any Maven command, check the repo root for a wrapper:
```bash
test -f ./mvnw && echo wrapper
```
If `mvnw`/`mvnw.cmd` exists, use `./mvnw` (or `mvnw.cmd` on Windows) for every command
below instead of a global `mvn` — many repos don't have Maven on `PATH` at all, only the
wrapper. Fall back to `mvn` only if no wrapper is present.

### Step 1: Verify Clean Git State

```bash
git status
```
If uncommitted changes exist, stop immediately:
```json
{ "status": "failed", "reason": "uncommitted changes — cannot safely upgrade" }
```

### Step 2: Baseline Verification

From the reactor root:
```bash
mvn clean compile -DskipTests
```
If this fails, halt before touching anything and report it as `pre-existing` — do not
attribute it to any package in the plan.

### Step 3: Process Each Entry in Plan Order

For each item in `plan.upgrades` and `plan.groups`:

**3a. Apply the version change** at the location `declaredVia`/`pom`/`property` already
name:
- `property` — update the `<properties>` value in the named `pom`.
- `direct` — update the inline `<version>` tag on that `<dependency>`.
- `bom` — update the version of the imported BOM coordinate itself in
  `<dependencyManagement>`; this is why BOM-driven changes arrive as a `group`, not a
  single package.
- `plugin` — update the `<version>` under `<build>/<plugins>`, preserving any
  `<configuration>` block unchanged.

Prefer the Versions Maven Plugin over hand-editing XML when it covers the case —
it's less error-prone:
```bash
mvn versions:set-property -Dproperty=jackson.version -DnewVersion=2.15.4 -DgenerateBackupPoms=false
```
Always pass `-DgenerateBackupPoms=false` — without it the plugin drops a
`pom.xml.versionsBackup` file next to every pom it touches, which is a file outside the
plan entry's scope and would otherwise need a manual cleanup step every time.

Fall back to a direct XML edit only when the versions-plugin doesn't cover the shape
(e.g. bumping a BOM's own coordinate).

**3b. Resolve and check the dependency graph:**
```bash
mvn clean compile
mvn dependency:tree
```
Run `dependency:tree` without `-q` — quiet mode suppresses the tree output entirely,
which would make this check silently pass without ever having run.

If `maven-enforcer-plugin` is configured, also run `mvn enforcer:enforce` to catch
convergence rule violations.

**3c. Run validation** in `policy.validation.order` (Maven mapping: `compile` →
`lint` [checkstyle/spotbugs/pmd, whichever is configured] → `test` → `package`/`verify`).
Skip any step whose plugin isn't configured in the pom — do not add one.

**3d. On success — commit:**
```bash
# DISABLED FOR TESTING — do not add/commit. See "TESTING MODE" note above.
# Re-enable these two lines to restore real commit behavior:
# git add <every pom touched by this entry only>
# git commit -m "chore(deps): upgrade <groupId>:<artifactId> <from> → <to>"
```
While disabled: leave the successful change uncommitted and staged/unstaged as-is,
and record `"commit": null` in the result for this entry instead of a commit hash.
Because nothing is committed between entries in this mode, Step 1's clean-state check
only applies to the very start of the run — expect accumulating uncommitted diffs
across multiple plan entries during a test run.

When re-enabled: for groups, list each coordinate's version change in the commit body.
One commit per plan entry — never batch multiple upgrades into one commit.

**3e. On failure — classify and revert:**

Classify using exactly one of: `breaking-api`, `behavior-change`, `dependency-convergence`
(tree/enforcer conflict), `bom-conflict`, `plugin-incompatibility`, `checkstyle-violation`,
`platform-env`, `stale-cache` (`~/.m2` corruption), `registry-network`, `registry-auth`,
`unresolvable-version` (the requested version does not exist in the repository at all —
distinct from `registry-network`/`registry-auth`, where the repository itself is
unreachable or rejects credentials; this should be rare since the planning agent
pre-validates every version, but classify here if a bad version still reaches you),
`pre-existing`.

Revert only what this entry touched, back to the state Step 3 started from:
```bash
git checkout -- <every pom touched by this entry>
mvn clean
```
Return the classification and reason. Do not decide retry vs skip, and do not
automatically try `fallbacks` — only attempt a specific fallback version if the caller
re-invokes you with it explicitly.

A failed entry never stops the rest of the plan — continue to the next one.

---

### Step 4: Build and Return Result

```json
{
  "repo": "/absolute/path/to/repo",
  "language": "java",
  "projectType": "backend",
  "status": "partial",
  "upgraded": [ { "coordinate": "com.fasterxml.jackson.core:jackson-databind", "from": "2.15.2", "to": "2.15.4", "commit": "abc1234" } ],
  "failed": [ { "coordinate": "org.apache.commons:commons-lang3", "attempted": "3.14.0", "failureType": "behavior-change",
                "reason": "2 tests failed after upgrade", "fallbacksAvailable": ["3.13.0"] } ],
  "skipped": [ { "coordinate": "org.springframework:spring-core", "failureType": "dependency-convergence",
                 "reason": "enforcer convergence conflict — returned for policy decision" } ],
  "preExisting": [ "1 test was failing before any upgrade ran — not attributed to any package" ]
}
```
Status values: `success` (all upgraded), `partial` (some upgraded, some failed),
`failed` (nothing could be upgraded), `clean` (plan had no upgrades).

While testing mode is active (Step 3d disabled), `commit` will be `null` for every
`upgraded` entry instead of a hash — that's expected, not a bug.

---

## Intelligence layer

Apply judgment only where deterministic rules can't:
- **Failure cause analysis** — reading compile/test/enforcer output to decide the
  classification.
- **Pre-existing issue detection** — reasoning about whether a failure predates the
  upgrade (Step 2 baseline is the primary signal).
- **Registry vs code triage** — distinguishing `registry-network`/`registry-auth`
  (settings.xml credentials, unreachable repo) from a genuine code/behavior problem,
  since only the latter is worth further diagnosis.
- **Dirty state detection** — checking git status before returning, reverting if needed.

Does NOT: decide retry vs skip, decide which fallback to try next, or make package
classification/coupling decisions — those belong to the policy engine/orchestrator.

---

## Important constraints

- Facts only — report what happened, not what should happen next.
- Never apply a default for a missing policy key — refuse and name the key instead.
- Never skip validation — every version change must be validated before committing.
- Never modify test assertions, suppressions, or source behavior to force a pass.
- Always revert to the pre-attempt state on failure — never leave partial edits.
- Never leave the repo in a dirty state.
- One commit per plan entry (package or group) — never batched.
- A failed entry never stops the rest of the repo's plan.

  (The last two are intentionally suspended while TESTING MODE is active — see the note
  under ## Behavior. Restore normal commit behavior before any real run.)
