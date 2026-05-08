# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is (and isn't)

This is the **public** `anthropics/claude-code` repository. It is **not** the source code for the Claude Code CLI itself. It contains:

- `plugins/` — 13 official Claude Code plugins (commands, agents, skills, hooks) bundled into a marketplace
- `scripts/` — Issue-management automation invoked from GitHub Actions (Bun + TypeScript / Bash)
- `.github/workflows/` — Workflows that run those scripts and Claude Code itself against the issue tracker
- `.claude/commands/` — Slash commands (`/dedupe`, `/triage-issue`, `/commit-push-pr`) wired into the workflows
- `examples/` — Reference templates for `settings.json`, MDM-managed deployments, and hook scripts
- `.claude-plugin/marketplace.json` — Registry that exposes the bundled plugins as a marketplace
- `.devcontainer/` — Sandboxed devcontainer for running Claude Code itself
- `CHANGELOG.md` — User-facing changelog for the CLI (the canonical source of release notes)

There is no application build, no test suite, and no compiled artifact. Most "code" is markdown (commands, agents, skills) plus a handful of Bun scripts. Don't fabricate build/test/lint commands — there are none at the repo root.

## Plugins (`plugins/`)

Every plugin follows the same shape (most pieces are optional):

```
plugins/<plugin-name>/
├── .claude-plugin/plugin.json   # name, description, version, author
├── commands/*.md                # slash commands
├── agents/*.md                  # subagent definitions
├── skills/<skill>/SKILL.md      # agent skills
├── hooks/hooks.json (+ handlers/)
└── README.md
```

When adding or modifying a plugin:

1. Update `.claude-plugin/marketplace.json` at the repo root if you add/rename a plugin — that file is the marketplace manifest and is what surfaces the plugin to users.
2. Keep each plugin's `plugin.json` in sync with the marketplace entry (name, version, author).
3. Plugin commands are plain markdown with YAML frontmatter (`allowed-tools`, `description`, `argument-hint`) — see `.claude/commands/triage-issue.md` for a representative example.
4. Hooks live under `hooks/hooks.json` and reference handlers (Python or shell) in a sibling directory like `hooks-handlers/`.

`plugins/README.md` is the human-facing index and should be updated when the plugin set changes.

## Issue automation (`scripts/` + `.github/workflows/`)

The repo runs an autonomous issue-management pipeline. Understanding the flow matters because the scripts and workflows are tightly coupled.

**Scripts (Bun-executed TypeScript / Bash):**
- `scripts/issue-lifecycle.ts` — **Single source of truth** for lifecycle labels (`invalid`, `needs-repro`, `needs-info`, `stale`, `autoclose`), their timeouts, and their messages. Any change to lifecycle behavior starts here.
- `scripts/sweep.ts` — Cron-driven sweep that walks the issue tracker and applies/escalates lifecycle labels per `issue-lifecycle.ts`. Supports `--dry-run`.
- `scripts/lifecycle-comment.ts` — Posts the heads-up comment when a lifecycle label is applied.
- `scripts/auto-close-duplicates.ts` / `backfill-duplicate-comments.ts` — Close-on-react and backfill helpers for the dedupe pipeline.
- `scripts/gh.sh` — **Locked-down `gh` wrapper.** Only allows `issue view|list`, `search issues`, `label list` and a fixed set of flags. Slash commands invoked from CI must use this — not raw `gh` — so the model can't run arbitrary GitHub commands.
- `scripts/edit-issue-labels.sh` — Adds/removes labels; reads the issue number from `GITHUB_EVENT_PATH` (not from arguments) to bind the action to the triggering event.
- `scripts/comment-on-duplicates.sh` — Posts the duplicates comment; same event-bound issue-number pattern.

**Slash commands invoked by workflows:**
- `.claude/commands/triage-issue.md` → `claude-issue-triage.yml` (on issues opened / commented). Adds category + lifecycle labels only — never comments.
- `.claude/commands/dedupe.md` → `claude-dedupe-issues.yml` (on issues opened). Five parallel search agents → filter agent → comment via `comment-on-duplicates.sh`.
- `.claude/commands/commit-push-pr.md` → branch + commit + push + `gh pr create` in a single message (used interactively, not in CI).

**Why the wrappers exist:** Scripts in `scripts/` enforce per-event capability caps (see `CLAUDE_CODE_SCRIPT_CAPS` in workflows, e.g. `'{"edit-issue-labels.sh":2}'`) and bind mutating actions to the workflow's event payload. When extending automation, add new wrappers in the same shape rather than calling `gh` directly from a slash command.

**Conventions:**
- Bun is the runtime for `.ts` scripts (`#!/usr/bin/env bun`). No `package.json` / `tsconfig.json` — scripts are standalone.
- `set -euo pipefail` is used in every shell script; preserve it.
- Workflows pin actions to commit SHAs (e.g. `actions/checkout@11bd71901bbe…`) — keep that pattern when adding steps that touch secrets.

## Examples (`examples/`)

- `examples/settings/` — `settings-lax.json`, `settings-strict.json`, `settings-bash-sandbox.json`. Reference policies for org-wide deployments. Some keys (`strictKnownMarketplaces`, `allowManagedHooksOnly`, `allowManagedPermissionRulesOnly`) only take effect at the enterprise tier.
- `examples/mdm/` — Templates for shipping managed settings via Jamf / Kandji / Intune / Group Policy. All templates encode the same minimal policy (`permissions.disableBypassPermissionsMode`) so they stay in lockstep.
- `examples/hooks/bash_command_validator_example.py` — Reference PreToolUse hook.

These are user-facing examples; warning callouts in their READMEs note they are community-maintained. Don't introduce divergent policies between the MDM templates without a reason.

## Working in this repo

- Don't add a build system, lockfile, or test runner unless asked — the repo is intentionally a collection of standalone artifacts.
- When adding a plugin, mirror the structure of an existing similar plugin (e.g. `code-review` for command+agents, `security-guidance` for hooks-only, `frontend-design` for skill-only) rather than inventing a new layout.
- The `CHANGELOG.md` is the public CLI changelog — do not edit it for repo-internal changes (plugin/script changes don't belong there).
- `LICENSE.md` is the SDK Source-Available license; new plugins inherit it and don't need their own license file.
