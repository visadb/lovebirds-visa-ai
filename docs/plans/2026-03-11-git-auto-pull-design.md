# Git Auto-Pull Design

## Overview

Add automatic `git pull` before loading the people database, mirroring the existing auto-push after save. A single `--no-git` flag replaces `--no-git-push` to disable all git operations (both pull and push).

## Git pull logic

A new `_git_pull(people_file: str)` function in `main.py`, called right before `load_people()`:

1. Derive `repo_dir` from `people_file`
2. Check if inside a git repo (`git rev-parse --is-inside-work-tree`) — if not, return silently
3. Check if tracking branch exists (`git rev-parse --abbrev-ref --symbolic-full-name @{u}`) — if not, return silently
4. Run `git pull` with interactive retry on failure (loop + `input()`, matching the push pattern in `config.py`)

## Flag change

Rename `--no-git-push` to `--no-git` (dest: `no_git`). This single flag disables both the pull before load and the push after save.

## Changes

- **Modified:** `src/lovebirds/cli/main.py` — Add `_git_pull()`, call before `load_people()` gated on `not args.no_git`, rename flag
- **Modified:** `src/lovebirds/cli/config.py` — Change `args.no_git_push` to `args.no_git`
- **Modified:** `tests/cli/test_config.py` — Update `no_git_push` to `no_git` in test helpers
- **New:** `tests/cli/test_main.py` — Tests for `_git_pull()`
