# Automatic Git Commit & Push for Database Changes

## Overview

Whenever the database file is modified and saved, if the file is inside a git repository, lovebirds automatically commits the database file and pushes it. Commit messages identify the subcommand and event but contain no personally identifiable information. The feature is on by default and can be disabled with `--no-git-push`.

## Git detection and opt-out

- Detect if the database file is in a git repo by running `git -C <dir> rev-parse --is-inside-work-tree`
- If not in a git repo, skip silently
- A `--no-git-push` flag on the root parser disables the auto commit+push
- Stored as `args.no_git_push` (default `False`)

## Commit message format

Built from the subcommand name and event ID (when available). No PII.

Examples:
- `lovebird: phase add (event: summer2026)`
- `lovebird: register (event: summer2026)`
- `lovebird: reformat`
- `lovebird: send (event: summer2026)`
- `lovebird: resolve-refs`

The subcommand name comes from the argparse subparser. The event ID comes from `Config.event_id` (already `None` when not applicable).

## Git operations and error handling

After `save_people()` succeeds in `Config.save_people()`:

1. **Check** if git is applicable: not `dry_run`, not `no_git_push`, file is in a git repo
2. **`git add <people_file>`** — stage only the database file
3. **`git commit <people_file> -m <message>`** — commit only the database file, avoiding other staged files. If nothing changed (exit code 1), log info and skip push
4. **`git push`** — push to the default remote. On failure, prompt the user to retry interactively (matching the existing retry pattern for save failures)

All git commands use `cwd` set to the directory containing the database file.

## Changes summary

### Modified: `cli/config.py`

- New private method `_git_commit_and_push()` — handles git add, commit, push with retry
- Modified `save_people()` — calls `_git_commit_and_push()` after successful save

### Modified: `cli/main.py`

- Add `--no-git-push` flag to the root argument parser

### No changes needed

- `io.py` — unchanged
- All subcommand modules — unchanged, they call `config.save_people()` as before
