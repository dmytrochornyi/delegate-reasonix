---
name: delegate-reasonix
description: Delegates heavy coding, deep reasoning, or file-refactoring tasks to Reasonix (a DeepSeek-powered coding agent). Use this skill whenever the user explicitly requests to run Reasonix, hand off a task to DeepSeek, or optimize token usage for heavy-lifting implementation. Also use it automatically right after the user approves an implementation plan - Claude plans, Reasonix executes, Claude audits.
argument-hint: "<prompt_for_reasonix>"
---

# Delegate to Reasonix

You are handing a task to Reasonix, an external DeepSeek-powered coding agent. Your job is to:
1. gather context
2. build a precise brief
3. run Reasonix
4. audit what it changed

Do not implement the task yourself.

The user's request is:

$ARGUMENTS

If the request is empty, stop and ask the user what Reasonix should do. Do not run anything.

## When invoked after an approved plan
- The task is the plan. Put the plan file's absolute path in the brief, plus its key steps verbatim.
- Delegate one phase (or a small group of related phases) per Reasonix run, in order. Audit each run before starting the next.
- Stop and report if a run fails, its tests fail, or its edits contradict the plan.

## Step 1: Inspect the repository

Use your native file tools (Read, Glob, Grep). Use Bash only for read-only git commands. Look only at what the task needs:

1. Run `git status --short` and `git log --oneline -5`. Note the current branch, any uncommitted work, and whether the repo has any commits yet.
2. Find the files, modules, functions, and tests the request touches. Prefer targeted Grep/Glob searches over reading whole directories.
3. Find the build and test commands (from `Cargo.toml`, `package.json`, `Makefile`, etc.).

Then give the user a short summary of 5 lines or fewer:
- the relevant files, as `path:line` where useful
- the build/test command
- any risks, such as uncommitted changes Reasonix might overwrite

If there is uncommitted work in files Reasonix will touch, warn the user before continuing.

## Step 2: Build the brief

Compose one prompt for Reasonix with this structure:

    Task: <the user's request, verbatim>

    Relevant files:
    - <path> — <one-line reason>
    - ...

    Constraints:
    - Only modify files needed for this task.
    - Build/test with: <command>
    - Run the tests after making changes and report the result.
    - When done, commit your work with git: stage only the files you changed
      (`git add <paths>`, never `git add -A` or `git add .`) and run
      `git commit -m "<short imperative summary>"`. One commit per run.
      Do not push, amend, rebase, reset, or change branches.

If the task itself is a git operation (e.g. "commit everything"), the task's wording wins over the staging rule above.

Keep the user's original wording intact. Add only paths and constraints. Do not reinterpret the task.

## Step 3: Take a baseline, then run Reasonix

Reasonix runs in yolo mode (no approval prompts), so a stray `git reset --hard`, `git checkout .` or `rm` could destroy uncommitted work. Before every run, make sure the working tree is clean:

- If `git status --porcelain` shows uncommitted changes, ask the user once: "Commit your current changes as a checkpoint before Reasonix runs?" On yes, run `git add -A && git commit -m "Checkpoint before Reasonix run"` (with the usual attribution line). On no, stop. Do not run Reasonix on a dirty tree.
- If the repo has no commits yet, the checkpoint becomes the initial commit.

Then record the repository state, so the changes can be attributed to Reasonix afterwards:

    mkdir -p "${TMPDIR:-/tmp}/reasonix" && touch "${TMPDIR:-/tmp}/reasonix/start_marker"
    git status --porcelain > "${TMPDIR:-/tmp}/reasonix/status_before.txt"
    git rev-parse HEAD > "${TMPDIR:-/tmp}/reasonix/head_before.txt" 2>/dev/null || echo none > "${TMPDIR:-/tmp}/reasonix/head_before.txt"

Then run Reasonix with the Bash tool. Notes on the command:
- Bare `reasonix "<task>"` opens the interactive terminal UI, which the Bash tool cannot drive. Always use the one-shot `reasonix run` subcommand.
- Pass the brief through a quoted heredoc so quotes, backticks, and `$` inside it are not interpreted by the shell.
- Close stdin (`< /dev/null`) so Reasonix can't hang waiting for input.
- A headless run has no approver, so it runs with `--yolo` (alias for `--permission-mode bypassPermissions`): no prompts, but Reasonix's sandbox, network policy and deny rules still apply. The clean-tree checkpoint above is what makes this safe; never skip it.
- If the repo has a remote (`git remote` prints anything), warn the user that yolo mode could push, and ask whether to continue before running.

    reasonix run --model deepseek-pro --dir "$PWD" --yolo "$(cat <<'REASONIX_PROMPT'
    <the brief from Step 2>
    REASONIX_PROMPT
    )" < /dev/null

Rules:
- Run it in the foreground with a timeout of 600000 ms. If the task is clearly larger than that, use `run_in_background` and wait for the completion notice. Do not poll.
- Run it once. If it exits non-zero, show the user the tail of the output and stop. Do not retry automatically.
- If Reasonix reports that the folder is not trusted or that its commands were refused, tell the user to run `! reasonix trust` once, then re-run this skill.
- If the audit shows Reasonix broke things, the rollback is `git reset --hard <head_before>`. Suggest it to the user; don't run it yourself.

## Step 4: Audit the result

After Reasonix exits, check what actually changed. Do not trust its own summary.

1. Run `git status --porcelain` and compare it with `status_before.txt`. Anything left uncommitted means Reasonix edited files it didn't commit; flag it.
2. Inspect Reasonix's commits against `head_before.txt`:
   - Run `git log --oneline <head_before>..HEAD`, then `git show --stat` and `git diff <head_before>..HEAD`. If `head_before` is `none`, use `git show --stat HEAD` and `git log --oneline`.
   - Confirm it stayed on the same branch, made no pushes, and didn't rewrite earlier history (`<head_before>` must still be an ancestor of HEAD: `git merge-base --is-ancestor <head_before> HEAD`).
3. Inspect any remaining uncommitted changes:
   - Tracked files: run `git diff --stat`, then `git diff` on the changed files.
   - Untracked files, or a repo with no commits yet: run `find . -newer "${TMPDIR:-/tmp}/reasonix/start_marker" -type f -not -path './.git/*' -not -path './target/*' -not -path './node_modules/*'`, then Read the changed files.
4. Check whether tests ran. Look in the Reasonix output for a test command and its result. If none ran, run the project's test command yourself and report the real outcome.

## Step 5: Report to the user

Keep the report short:
- **Commits:** each new commit's hash and message
- **Changed:** each file, with a one-line description of what changed
- **Tests:** the command that was run, and pass/fail with counts, or "not run"
- **Concerns:** edits outside the task's scope, deleted code, a failing build, TODOs left behind, or anything that contradicts the brief

Do not push, revert, amend, or reset Reasonix's commits unless the user asks. If the audit finds a problem, report it and let the user decide (e.g. `git revert` or another Reasonix run).
