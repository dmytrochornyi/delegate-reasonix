# delegate-reasonix

[Русская версия](README.ru.md)

A [Claude Code](https://claude.com/claude-code) skill that splits work between two agents:
**Claude plans and audits, Reasonix (a DeepSeek-powered coding
agent) writes the code.** Claude does the reasoning, Reasonix does the bulk implementation, which saves
Claude tokens on large tasks.

## How it works

When invoked (`/delegate-reasonix <task>`, or automatically after you approve an implementation plan),
Claude:

1. **Inspects the repo**: git state, the files and tests the task touches, the build/test command.
   It warns you about uncommitted work Reasonix could overwrite.
2. **Builds a brief**: your task verbatim, the relevant files, and constraints. Reasonix may only touch
   what the task needs, must run the tests, and must commit its own work (staging only its files,
   no push, amend, rebase or reset).
3. **Takes a baseline and runs Reasonix** headlessly:
   `reasonix run --model deepseek-pro --dir "$PWD" --yolo "<brief>" < /dev/null`.
   Before that, the working tree must be clean. Claude offers a checkpoint commit, because yolo mode
   has no approval prompts. If the repo has a remote, Claude asks before running.
4. **Audits the result** without trusting Reasonix's own summary: new commits, diff, history not
   rewritten, leftover uncommitted files, and whether tests actually ran (it runs them if not).
5. **Reports**: commits, changed files, test results, and concerns. It never reverts or pushes on its
   own; if something broke it suggests `git reset --hard <head_before>` and lets you decide.

With an approved plan, it delegates one phase at a time and audits each run before the next.

## Requirements

- Claude Code
- Reasonix CLI installed and configured with your DeepSeek API key (the key stays in Reasonix's own
  config and is never passed through this skill)
- A git repository to work in

## Installation

```bash
git clone https://github.com/dmytrochornyi/delegate-reasonix.git
mkdir -p ~/.claude/skills
cp -r delegate-reasonix/skills/delegate-reasonix ~/.claude/skills/
```

Restart Claude Code. The skill appears as `/delegate-reasonix`.

If Reasonix refuses to run commands in a folder, trust it once: `reasonix trust`.

## Usage

```
/delegate-reasonix Add a --json flag to the export command and cover it with tests
```

Or let it trigger after a plan: approve the plan, and Claude hands each phase to Reasonix.

To change the model, edit `--model deepseek-pro` in `SKILL.md`.

## Safety notes

- Reasonix runs with `--yolo` (no approval prompts). Reasonix's own sandbox, network policy and deny
  rules still apply, and the clean-tree checkpoint makes every run reversible.
- The skill never pushes, and asks before running in a repo that has a remote.

## License

[MIT](LICENSE)
