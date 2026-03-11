# Git Auto-Pull Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add automatic `git pull` before loading the database, and consolidate `--no-git-push` into a single `--no-git` flag.

**Architecture:** New `_git_pull()` function in `main.py` called before `load_people()`. Rename `--no-git-push` → `--no-git` everywhere. Same repo/tracking-branch detection pattern as `_git_commit_and_push()`.

**Tech Stack:** Python 3.13+, pytest + unittest.mock

**Design doc:** `docs/plans/2026-03-11-git-auto-pull-design.md`

---

### Task 1: Rename `--no-git-push` to `--no-git`

**Files:**
- Modify: `src/lovebirds/cli/main.py:111-117`
- Modify: `src/lovebirds/cli/config.py:55`
- Modify: `tests/cli/test_config.py` (6 occurrences of `no_git_push`)

**Step 1: Update the flag in `main.py`**

Change lines 111-117 from:

```python
    root.add_argument(
        "--no-git-push",
        dest="no_git_push",
        default=False,
        action="store_true",
        help="Don't automatically commit and push the database file after saving.",
    )
```

To:

```python
    root.add_argument(
        "--no-git",
        dest="no_git",
        default=False,
        action="store_true",
        help="Don't automatically pull before loading or commit and push after saving the database file.",
    )
```

**Step 2: Update `config.py`**

Change line 55 from:

```python
        if not self.args.no_git_push:
```

To:

```python
        if not self.args.no_git:
```

**Step 3: Update all test references in `tests/cli/test_config.py`**

Replace all 6 occurrences of `no_git_push` with `no_git`:

- Line 18: `no_git_push=True,` → `no_git=True,`
- Line 61: `test_no_git_push_flag_skips_git` → `test_no_git_flag_skips_git`
- Line 65: `no_git_push=True,` → `no_git=True,`
- Line 79: `no_git_push=False,` → `no_git=False,`
- Line 93: `no_git_push=False,` → `no_git=False,`
- Line 113: `no_git_push=False,` → `no_git=False,`

**Step 4: Run tests**

Run: `poetry run pytest -q`
Expected: All pass

**Step 5: Commit**

```bash
git add src/lovebirds/cli/main.py src/lovebirds/cli/config.py tests/cli/test_config.py
git commit -m "refactor: rename --no-git-push to --no-git"
```

---

### Task 2: Add `_git_pull()` and wire it in

**Files:**
- Modify: `src/lovebirds/cli/main.py:315-316`
- Create: `tests/cli/test_main.py`

**Step 1: Write the tests**

Create `tests/cli/test_main.py`:

```python
import subprocess
from unittest.mock import patch, call

import pytest

from lovebirds.cli.main import _git_pull


class TestGitPull:
    @patch("lovebirds.cli.main.subprocess.run")
    def test_not_in_git_repo_skips(self, mock_run):
        mock_run.side_effect = subprocess.CalledProcessError(128, "git")
        _git_pull("/tmp/test.yaml")
        assert mock_run.call_count == 1

    @patch("lovebirds.cli.main.subprocess.run")
    def test_no_tracking_branch_skips(self, mock_run):
        def side_effect(*args, **kwargs):
            cmd = args[0]
            if "@{u}" in cmd:
                return subprocess.CompletedProcess(args=[], returncode=128)
            return subprocess.CompletedProcess(args=[], returncode=0)

        mock_run.side_effect = side_effect
        _git_pull("/tmp/test.yaml")
        # Only rev-parse (repo check) and rev-parse (tracking branch check)
        assert mock_run.call_count == 2
        pull_calls = [c for c in mock_run.call_args_list if "pull" in c[0][0]]
        assert len(pull_calls) == 0

    @patch("lovebirds.cli.main.subprocess.run")
    def test_pulls_successfully(self, mock_run):
        mock_run.return_value = subprocess.CompletedProcess(args=[], returncode=0)
        _git_pull("/tmp/test.yaml")
        calls = mock_run.call_args_list
        # 1: rev-parse (repo), 2: rev-parse (tracking), 3: pull
        assert len(calls) == 3
        assert "--is-inside-work-tree" in calls[0][0][0]
        assert "@{u}" in calls[1][0][0]
        assert "pull" in calls[2][0][0]
```

**Step 2: Run tests to verify they fail**

Run: `poetry run pytest tests/cli/test_main.py -v`
Expected: FAIL — `_git_pull` not found

**Step 3: Implement `_git_pull()` in `main.py`**

Add these imports at the top of `main.py` (some already exist):

```python
import subprocess
import traceback
```

Add the function before `parse_arguments()`:

```python
def _git_pull(people_file: str) -> None:
    repo_dir = os.path.dirname(os.path.abspath(people_file))

    # Check if inside a git repo
    try:
        subprocess.run(
            ["git", "-C", repo_dir, "rev-parse", "--is-inside-work-tree"],
            check=True,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
        )
    except subprocess.CalledProcessError:
        return

    # Only pull if the current branch has a remote tracking branch
    result = subprocess.run(
        ["git", "-C", repo_dir, "rev-parse", "--abbrev-ref", "--symbolic-full-name", "@{u}"],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
    )
    if result.returncode != 0:
        return

    # Pull with interactive retry
    while True:
        try:
            subprocess.run(
                ["git", "-C", repo_dir, "pull"],
                check=True,
            )
            break
        except subprocess.CalledProcessError:
            traceback.print_exc()
            input("Git pull failed. Press enter to retry.")
```

**Step 4: Wire it into `parse_arguments()`**

Change lines 315-316 from:

```python
    # Load people
    people = load_people(args.people_file)
```

To:

```python
    # Pull latest changes before loading
    if not args.no_git:
        _git_pull(args.people_file)

    # Load people
    people = load_people(args.people_file)
```

**Step 5: Run tests**

Run: `poetry run pytest -q`
Expected: All pass

**Step 6: Run mypy and black**

Run: `poetry run mypy src/ && poetry run black src/ tests/`
Expected: Clean

**Step 7: Commit**

```bash
git add src/lovebirds/cli/main.py tests/cli/test_main.py
git commit -m "feat: auto git pull before loading database"
```
