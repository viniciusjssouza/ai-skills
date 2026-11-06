---
name: prepare-for-pr
description: Prepare the current branch for opening a PR — review the branch's changes for missing observability (logs and metrics) and offer to stop and add them, fix PMD issues on the branch's changes, run the local pre-commit checks, and amend the resulting changes into the last commit.
---

Get the current branch ready for a PR so CI doesn't trip on avoidable issues. Run the steps in order and stop
at the first one that fails, reporting why.

Invoking this skill is the user's explicit authorization to **amend the last commit** (step 5). It is not an
authorization to create new commits, push, or force-push.

## 0. Preconditions

- Read `~/.claude/CODE_GUIDELINES.md` in full before changing any code; fixes must follow it.
- Work from the repository root (`git rev-parse --show-toplevel`).
- The base branch is `main` unless the repo's instructions say otherwise (e.g. `clpr` in `clpr-hiero`).
- Stop if the current branch *is* the base branch, or if HEAD has no commits beyond the base
  (`git log --oneline <base>..HEAD` is empty) — there is nothing to amend.
- Note any uncommitted changes that already exist before you start and tell the user; they will be folded into
  the amend in step 5, so confirm that is intended before continuing.

## 1. Observability review (branch changes only)

Check whether the branch's changes can be diagnosed in production from logs and metrics alone, and suggest what
is missing. This step only **suggests**; it changes no code by itself.

1. Read the branch's changes: `git diff <base>` (committed + uncommitted), production code only — skip tests and
   test fixtures. For each changed method, read enough of the surrounding code and of its callers to know how a
   failure surfaces: where exceptions are caught, at what level they are logged, and what the caller logs.
2. Look for decision points in the **changed** code that an operator could not reconstruct from the existing
   output:
   - Rejections and failures that are swallowed, translated into a status or result code, or logged only at
     DEBUG, so they leave no trace at the default level.
   - Failure messages that lack the context needed to tell which input failed (ids, slots, versions, sizes).
   - State transitions and security-relevant decisions (trust, keys, configuration, endpoints, rotations) with no
     INFO log, or with one that omits what was accepted.
   - Silent fallbacks and defaults (a missing value replaced by zero, empty, or "absent"; a branch taken without a
     trace).
   - Logs whose level does not match their impact (guidelines §3: ERROR only for unrecoverable conditions, WARN
     pages someone, INFO for transitions, DEBUG for diagnostics; library code one level lower), and log lines that
     claim something the code does not do.
   - Counts, rates, and latencies worth a metric: outcomes per type (accepted / rejected by reason), sizes,
     durations of expensive operations, and anything a dashboard or alert would watch. Use the metrics facility
     the module already uses; if the touched area has no metrics yet, still suggest them, and say they need
     wiring.
3. Skip what the code already covers: an exception whose message names the reason does not need a log at the
   throw site if the catch site logs it at a visible level. Do not suggest logging in hot paths at INFO or above,
   and do not suggest logging secrets or full payloads.
4. Present the suggestions as a list ordered by priority (incident/security impact first, then traceability, then
   DEBUG-only additions). Each suggestion must be concrete enough to implement as written:
   - location as `file_path:line_number`;
   - for a log: the level, the message, and the fields it carries (or the existing log line to change, and how);
   - for a metric: name, type (counter / gauge / histogram), what increments or sets it, and any labels;
   - one sentence on why: what question it answers during an incident.
   If nothing is missing, say so in one line and continue with step 2 without asking.
5. Otherwise, ask the user (AskUserQuestion) to choose:
   - **Stop and add observability** — end the skill here, before any PMD fix, check, or amend, so the suggested
     changes can be added (by the user, or by Claude on a follow-up request that names which suggestions to
     apply). The user re-runs the skill afterwards.
   - **Continue without adding it** — keep going with step 2; list the suggestions again in the final report under
     "Observability (not applied)".

## 2. PMD fixes (branch changes only)

Follow `~/.claude/skills/pmd-fix/SKILL.md`, with these specifics:

1. Find the ruleset inside the project (e.g. `config/pmd/ruleset.xml`).
2. Collect the changed Java files that still exist:
   `git diff --name-only <base>` (committed + uncommitted), keep `*.java` that exist on disk.
3. Run `pmd check -R <ruleset> -d <comma-separated files> -f text --no-progress` into the session scratchpad.
4. **Keep only findings on lines this branch changed**: parse `git diff -U0 <base>` hunks (`@@ ... +start,count @@`)
   into per-file line sets and filter the PMD output (`path:line:\tRule:\tmessage`) by them.
5. Fix what makes sense; skip what doesn't. Default skip list (conflicts with the guidelines or is noise here):
   - `GuardLogStatement` (don't wrap info/warn/error — or parameterized debug — logs in conditionals)
   - `CommentDefaultAccessModifier` (no `/* default */`)
   - ConcurrentHashMap-outside-concurrency rules
   - `UseExplicitTypes` (`var` is used per guidelines §11)
   - `MethodNamingConventions` on tests (given/when/then names per §15)
   - `FieldNamingConventions` for `logger` (§9 lowercase logger)
   - `OnlyOneReturn`, `UnitTestContainsTooManyAsserts`, `UnitTestShouldIncludeAssert` (Mockito `verify` counts),
     `TooManyStaticImports`, `TooManyMethods`, `ExcessiveImports`, `AvoidUsingHardCodedIP` in tests
   - `InvalidLogMessageFormat` when the extra argument is a trailing `Throwable` (Log4j logs it as the stack trace)
   - `AvoidFieldNameMatchingMethodName` for record-style accessors
   - Findings on pre-existing code that was only moved or renamed — mention, don't refactor (§21)

   Usually worth fixing: missing Javadoc on public constructors/methods (§19), `UnnecessaryModifier`,
   `AssignmentInOperand` (prefer `for (x = next(); x != null; x = next())`), `AvoidDuplicateLiterals` when a
   constant already exists, `MethodArgumentCouldBeFinal` (§8), `UseUnderscoresInNumericLiterals` for sizes/amounts
   (not ports), catching broad `Exception` (§2).
6. Ask the user about anything ambiguous instead of guessing.
7. Run `./gradlew spotlessApply` and the unit tests of the affected module(s) for the touched packages
   (e.g. `./gradlew :app:test --tests 'com.example.pkg.*'`). All must pass before continuing.

## 3. Local pre-commit checks

1. Run `pre_commit --skipTests` from the repo root (installed at `~/bin/pre_commit`). It is long-running: run it in
   the background and wait for the completion notification.
2. If it fails, fix the cause, rerun it, and repeat until it passes. Never weaken tests or checks to get a pass.
3. `pre_commit --skipTests` runs `./gradlew spotlessApply` and then the CI-style checks
   (`:app:javadoc checkAllModuleInfo checkVersionConsistency validatePomFiles`). The script has no `set -e`, so its
   exit code is only that of the last command: read the output and treat a failure in either step as a failure.

## 4. Review what will be amended

- Show `git status --short` and `git diff --stat`.
- Only stage files changed by steps 2–3 (plus pre-existing uncommitted changes the user confirmed in step 0).
  Never stage untracked files that aren't part of the work (scratch notes, local artifacts, `*.txt` dumps, etc.).
- If nothing changed, skip step 5 and say the branch was already clean.

## 5. Amend the last commit

1. Check whether HEAD is already pushed: if `git branch -r --contains HEAD` lists any branch, **stop and ask** —
   amending a pushed commit requires a force-push, which the user must decide on.
2. Stage the files from step 4 explicitly by path (`git add <paths>`), not `git add -A`.
3. `git commit --amend --no-edit`.
   - Let the repo's hooks run: never use `--no-verify` or `--no-gpg-sign` (GPG signing and the DCO
     `Signed-off-by:` trailer must survive).
   - Do not add attribution lines (e.g. `Co-Authored-By`), in line with the repo's instructions.
4. Do not push.

## 6. Report

Keep it short:
- Observability: "no gaps found", or the suggestions the user chose not to apply (under "Observability (not
  applied)").
- PMD: what was fixed, what was skipped and why (grouped by rule), anything left for the user.
- Tests run and their result.
- `pre_commit` result: spotless and the CI-style checks.
- The amended commit (`git log --oneline -1`) and the files folded into it — or that nothing needed amending.
