# rstack

rstack is [@michaelshimeles](https://github.com/michaelshimeles)'s agent development stack: seven [agent skills](https://code.claude.com/docs/en/skills) and an [`AGENTS.md`](AGENTS.md) workflow that tells a coding agent how to take a change from a clean branch to a reviewed pull request. It is written for Claude Code, Cursor, Codex, and other agents that load `SKILL.md` folders.

[![skills.sh](https://skills.sh/b/michaelshimeles/rstack)](https://skills.sh/michaelshimeles/rstack)

Use it if you want an agent to work in its own worktree, follow your project's architecture, prove each change with evidence, and stop only when Greptile rates the PR 5/5. Copy `AGENTS.md` into another repo and swap in that repo's checks.

## The workflow

[`AGENTS.md`](AGENTS.md) picks the steps that fit each task. Read-only reviews and investigations skip all of them. Small documentation edits get an isolated branch and content checks, with no runtime evidence or bot review.

1. **Isolate.** `/new-feature` puts the edit in its own Git worktree and branch from `origin/main`, after checking open PRs for overlap. Nothing gets built on `main`.
2. **Build.** `/code-structure` applies when extracting shared operations or choosing boundaries. The project's existing architecture wins. The service layer pattern is the default only when there is none.
3. **Prove.** Run the relevant checks. `/evidence-driven-testing` records the failure before the fix and the success after it, for any change in runtime behavior.
4. **Ship.** Open the PR with evidence that fits the change. `/before-and-after` adds a screenshot table when visuals help. `/greploop` runs Greptile review cycles for code and executable workflow changes until the current commit scores 5/5 with no unresolved actionable findings, or until it hits the iteration cap, a timeout, or a missing integration. It then reports the real score.

`/unslop` runs over anything a person will read along the way: commit messages, the PR title and body, docs, and code comments.

`AGENTS.md` also sets multi-agent rules: one worktree and branch per task and agent, no commits to `main`, `--force-with-lease` only on your own branch, and no merge unless someone asks for it.

## Skills

| Skill | What it does |
|---|---|
| [`new-feature`](new-feature/SKILL.md) | Isolate repository edits in a Git worktree before starting a feature, fix, or documentation change. Preserve a worktree already assigned to the same task. |
| [`code-structure`](code-structure/SKILL.md) | Use when multiple workflows duplicate the same operational logic, or when deciding what belongs in actions vs shared services. |
| [`evidence-driven-testing`](evidence-driven-testing/SKILL.md) | Records visual proof while testing UI behavior, with an annotated screen recording, scripted captures for headless runs, and measured output for non-UI changes. |
| [`before-and-after`](before-and-after/SKILL.md) | Captures before/after screenshots of web pages or elements for visual comparison and outputs a PR-ready markdown table. |
| [`greploop`](greploop/SKILL.md) | Address Greptile feedback on a PR, MR, or shelved changelist through bounded review and fix cycles. |
| [`greploop-apps`](greploop-apps/SKILL.md) | Run the greploop workflow with the `@greptile-apps` trigger when the normal trigger refuses a large PR. Requires `greploop`. |
| [`unslop`](unslop/SKILL.md) | Cut AI tells from text you write or edit for a human reader. Apply before committing, posting, or sending. |

### Skill notes

- **evidence-driven-testing.** The recorder, `scripts/evidence.py`, needs Python 3 and `ffmpeg`/`ffprobe` with `libx264` and the `ass` filter. It runs on Linux, macOS, and Windows with `doctor`, `start`, `annotate`, and `stop` commands, and writes `evidence.mp4`, `report.md`, and `manifest.json`. Run `python3 scripts/evidence.py doctor` from the skill folder to check your machine. The agent drives the app with computer use, or [cua-driver](https://github.com/trycua/cua) when it has no computer-use tools. The headless helper, `scripts/run-playwright.sh`, needs Node 20+ and npm and installs a pinned Playwright in a temp directory, outside your project.
- **before-and-after.** Drives the `@vercel/before-and-after` CLI. Install it with `npm i -g @vercel/before-and-after agent-browser`. The default upload host, 0x0.st, is public. Pass `--upload-url` for anything sensitive.
- **greploop.** Needs Greptile installed on the repo and an authenticated `gh`, `glab`, or `p4`. The GitHub helper, `scripts/github_review.py`, needs Python 3.10+. It records each trigger and rejects stale checks and summaries. The cap is `--max-iterations`, default 10.
- **greploop-apps.** Has no implementation of its own. Install it next to `greploop`.
- **unslop.** The body matches upstream. The frontmatter drops `disable-model-invocation: true` and scopes the description to text the agent writes or edits for people, so agents apply it without a typed `/unslop`. Put the flag back if you want slash-command-only behavior.

## Install

### With `npx skills`

The [`skills`](https://skills.sh) CLI installs to Claude Code, Cursor, Codex, and most other agents:

```bash
npx skills add michaelshimeles/rstack
```

Useful flags: `--list` shows the skills without installing, `-s <name>` picks skills, `-a <agent>` picks agents (`claude-code`, `cursor`, `codex`), and `-g` installs at user level instead of project level.

### By hand

Clone the repo and copy skill folders into your agent's skills directory. For Claude Code:

```bash
git clone https://github.com/michaelshimeles/rstack.git
cd rstack
cp -r new-feature code-structure evidence-driven-testing before-and-after greploop greploop-apps unslop ~/.claude/skills/
```

`greploop-apps` only works with `greploop` installed beside it.

### The workflow file

The skills work alone, but the workflow lives in `AGENTS.md`. Copy it into the root of the repo you want the agent to work in, and replace the "Checks in this repo" section with that repo's checks. Cursor and Codex read `AGENTS.md` from the repo root. Claude Code reads `CLAUDE.md`, so add a `CLAUDE.md` that contains `@AGENTS.md`.

Once installed, agents pick skills by their descriptions. In Claude Code you can also call one directly, for example `/new-feature` or `/greploop`.

## Repo layout

```
AGENTS.md                    workflow and multi-agent rules
new-feature/                 SKILL.md
code-structure/              SKILL.md
evidence-driven-testing/     SKILL.md, scripts/evidence.py, scripts/run-playwright.sh, scripts/playwright/
before-and-after/            SKILL.md, scripts/upload-and-copy.sh, scripts/adapters/, LICENSE
greploop/                    SKILL.md, scripts/github_review.py, references/, LICENSE
greploop-apps/               SKILL.md, LICENSE
unslop/                      SKILL.md, LICENSE
tests/                       pytest suite for the recorder, review helper, and Playwright runner
```

## Checks

Run `python3 -m pytest tests/ -q` and `git diff --check`. The tests need Python 3.10+, pytest, and `ffmpeg`/`ffprobe` with `libx264` and the `ass` filter. The Playwright runner tests also need Node and Bash, and they stub npm downloads. If you change that helper, run a real Chromium capture with the documented command too. The tests use synthetic recorder input and local API fixtures, so they don't cover live uploads, every native capture backend, or a live Greptile review.

## Adding a skill

1. Create a kebab-case folder named after the skill.
2. Add a `SKILL.md` with `name` and `description` frontmatter. Agents use the description to decide when the skill applies, so lead with the trigger ("Use when...").
3. Keep the instructions short. Move long material into reference files in the same folder.

## Sources and licenses

`new-feature`, `code-structure`, and `evidence-driven-testing` were written here. The rest are vendored, each with its license in its folder:

| Skill | Upstream | License |
|---|---|---|
| `before-and-after` | [vercel-labs/before-and-after](https://github.com/vercel-labs/before-and-after) | PolyForm Shield 1.0.0 |
| `greploop` | [greptileai/skills](https://github.com/greptileai/skills) | MIT |
| `greploop-apps` | local entrypoint derived from greptileai's greploop | MIT |
| `unslop` | [cursor/plugins (pstack)](https://github.com/cursor/plugins/tree/main/pstack/skills/unslop) | MIT |

The repo has no top-level license file. Only the vendored skills ship a license.
