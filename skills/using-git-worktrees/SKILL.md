---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from current workspace or before executing implementation plans - creates isolated git worktrees using the wt CLI tool
---

# Using Git Worktrees

## Overview

Git worktrees create isolated workspaces sharing the same repository, allowing work on multiple branches simultaneously without switching.

This skill uses [`wt`](https://github.com/timvw/wt), a Git worktree manager that handles placement, shell integration, and multi-repo coordination.

**Core principle:** Use `wt` for all worktree operations. It handles directory placement, duplicate prevention, and auto-cd.

**Announce at start:** "I'm using the using-git-worktrees skill to set up an isolated workspace."

## Prerequisites

`wt` must be installed with shell integration configured:

```bash
# Install
brew install timvw/tap/wt

# Configure shell integration (enables auto-cd and tab completion)
wt init
```

Verify configuration:

```bash
wt info
```

This shows the active strategy, pattern, root directory, and available variables.

## Creating a Worktree

```bash
# Create a new branch in a worktree (defaults to main/master as base)
wt create <branch-name>

# Create from a specific base branch
wt create <branch-name> develop
```

With shell integration, `wt create` automatically changes into the new worktree directory.

### Checkout Existing Branch

```bash
# Checkout an existing branch into a worktree
wt checkout <branch-name>
wt co <branch-name>          # short alias

# Interactive selection from available branches
wt co
```

### Checkout a PR or MR

```bash
# GitHub PR (requires gh CLI)
wt pr 123
wt pr https://github.com/org/repo/pull/123
wt pr                        # interactive selection

# GitLab MR (requires glab CLI)
wt mr 42
wt mr https://gitlab.com/org/repo/-/merge_requests/42
wt mr                        # interactive selection
```

## Multi-Repository Tasks

When a task requires changes across multiple repositories, `wt` can group worktrees by feature instead of by repo. This keeps all related code together.

### Configuration

Set up a custom pattern that puts the branch name first:

```toml
# ~/.config/wt/config.toml
strategy = "custom"
pattern = "{.worktreeRoot}/{.branch}/{.repo.Name}"
```

### Workflow

Use the same branch name in each repository:

```bash
cd ~/src/shared-lib
wt create feat/PROJ-123

cd ~/src/main-app
wt create feat/PROJ-123
```

This produces a grouped layout:

```
~/dev/worktrees/
  feat/PROJ-123/
    shared-lib/
    main-app/
```

All repos for one task are under a single directory, making cross-repo work straightforward.

### Alternative: Environment Variable Grouping

For tasks where branch names differ across repos, use an environment variable:

```toml
# ~/.config/wt/config.toml
strategy = "custom"
pattern = "{.worktreeRoot}/{.env.FEATURE}/{.repo.Name}"
```

```bash
export FEATURE=PROJ-42-new-checkout

cd ~/src/frontend
wt create main

cd ~/src/backend
wt create main
```

Switch to a different feature by changing the variable:

```bash
export FEATURE=PROJ-99-hotfix
```

## Project Setup via Hooks

Instead of manually detecting and running setup commands, configure `wt` hooks to automate dependency installation:

```toml
# ~/.config/wt/config.toml
[hooks]
# Copy environment files from main worktree
post_create = ["test -f $WT_MAIN/.env && cp $WT_MAIN/.env $WT_PATH/.env || true"]

# Install dependencies after creating or checking out a worktree
post_checkout = ["cd $WT_PATH && test -f package.json && npm install || true"]
```

Hook environment variables available: `$WT_PATH`, `$WT_BRANCH`, `$WT_MAIN`, `$WT_REPO_NAME`.

Common hook patterns:

| Project type | Hook command |
|-------------|-------------|
| Node.js | `cd $WT_PATH && npm install` |
| Python (uv) | `cd $WT_PATH && uv sync` |
| Python (poetry) | `cd $WT_PATH && poetry install` |
| Rust | `cd $WT_PATH && cargo build` |
| Go | `cd $WT_PATH && go mod download` |

Pre-hooks abort the operation on failure. Post-hooks warn but continue.

## Verify Clean Baseline

After creating the worktree, run tests to ensure a clean starting point:

```bash
# Use project-appropriate command
npm test
cargo test
pytest
go test ./...
```

**If tests fail:** Report failures, ask whether to proceed or investigate.

**If tests pass:** Report ready.

### Report Location

```
Worktree ready at <full-path>
Tests passing (<N> tests, 0 failures)
Ready to implement <feature-name>
```

## Managing Worktrees

```bash
# List all worktrees
wt list
wt ls

# Remove a worktree
wt remove <branch>
wt rm <branch>
wt rm                        # interactive selection
wt rm -f <branch>            # force remove (modified worktree)

# Clean up worktrees for merged branches
wt cleanup
wt cleanup --dry-run         # preview what would be removed

# Remove stale worktree administrative files
wt prune
```

## Quick Reference

| Situation | Action |
|-----------|--------|
| Start new feature | `wt create <branch>` |
| Work on existing branch | `wt checkout <branch>` or `wt co` |
| Review a GitHub PR | `wt pr <number>` |
| Review a GitLab MR | `wt mr <number>` |
| Multi-repo task | Same branch name + custom pattern |
| List worktrees | `wt list` |
| Remove worktree | `wt remove <branch>` |
| Clean merged branches | `wt cleanup` |
| Check configuration | `wt info` |
| Tests fail during baseline | Report failures + ask |

## Common Mistakes

### Forgetting shell integration

- **Problem:** No auto-cd after `wt create`, must manually navigate to worktree
- **Fix:** Run `wt init` and restart shell. Verify with `wt info`.

### Not configuring hooks for setup

- **Problem:** Dependencies not installed in new worktrees, tests fail due to missing packages
- **Fix:** Add `post_create`/`post_checkout` hooks in `~/.config/wt/config.toml`

### Different branch names across repos in multi-repo task

- **Problem:** Worktrees for the same task scattered across different directories
- **Fix:** Use consistent branch names, or use `{.env.FEATURE}` pattern variable

### Proceeding with failing tests

- **Problem:** Can't distinguish new bugs from pre-existing issues
- **Fix:** Report failures, get explicit permission to proceed

## Example Workflow

### Single Repository

```
You: I'm using the using-git-worktrees skill to set up an isolated workspace.

[Run: wt create feature/auth]
[Auto-cd to worktree]
[Run npm test - 47 passing]

Worktree ready at ~/dev/worktrees/myproject/feature/auth
Tests passing (47 tests, 0 failures)
Ready to implement auth feature
```

### Multi-Repository

```
You: I'm using the using-git-worktrees skill to set up isolated workspaces
     across multiple repositories.

[Run: cd ~/src/shared-lib && wt create feat/PROJ-123]
[Run: cd ~/src/main-app && wt create feat/PROJ-123]

Worktrees ready:
  ~/dev/worktrees/feat/PROJ-123/shared-lib/
  ~/dev/worktrees/feat/PROJ-123/main-app/

Ready to implement PROJ-123 across both repos.
```

## Red Flags

**Never:**
- Use raw `git worktree add` when `wt` is available
- Skip baseline test verification
- Proceed with failing tests without asking
- Use different branch names across repos in a multi-repo task without `{.env.*}` grouping

**Always:**
- Use `wt create` / `wt checkout` for worktree operations
- Verify clean test baseline after creation
- Use consistent branch names for multi-repo tasks
- Check `wt info` if unsure about current configuration

## Integration

**Called by:**
- **brainstorming** (Phase 4) - REQUIRED when design is approved and implementation follows
- **subagent-driven-development** - REQUIRED before executing any tasks
- **executing-plans** - REQUIRED before executing any tasks
- Any skill needing isolated workspace

**Pairs with:**
- **finishing-a-development-branch** - REQUIRED for cleanup after work complete
