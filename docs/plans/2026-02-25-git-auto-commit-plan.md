# Automatic Git Commit & Push — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Automatically git commit and push the database file after every save, with a PII-free commit message showing the subcommand and event ID.

**Architecture:** Git logic lives in `Config._git_commit_and_push()`, called at the end of `Config.save_people()`. Subcommand names are stored via `set_defaults(subcommand=...)` in `main.py`. A `--no-git-push` root flag opts out.

**Tech Stack:** Python 3.13+, subprocess (git CLI), pytest + pytest-mock for testing.

**Design doc:** `docs/plans/2026-02-25-git-auto-commit-design.md`

---

### Task 1: Add `--no-git-push` flag and `subcommand` defaults to `main.py`

**Files:**
- Modify: `src/lovebirds/cli/main.py`
- Test: `tests/cli/test_config.py` (indirectly — the flag just needs to exist on `args`)

**Step 1: Write the failing test**

Add to `tests/cli/test_config.py`:

```python
class TestConfigGitPush:
    @patch("lovebirds.cli.config.save_people")
    @patch("lovebirds.cli.config.check_type")
    def test_no_git_push_flag_skips_git(self, mock_check, mock_save):
        args = argparse.Namespace(
            dry_run=False,
            people_file="/tmp/test_people.yaml",
            no_git_push=True,
            subcommand="reformat",
        )
        config = Config(args=args, event=None, event_id="", people={})
        with patch.object(config, "_git_commit_and_push") as mock_git:
            config.save_people(backup=False)
        mock_git.assert_not_called()
```

**Step 2: Run test to verify it fails**

Run: `poetry run pytest tests/cli/test_config.py::TestConfigGitPush -v`
Expected: FAIL — `Config` has no `_git_commit_and_push` method

**Step 3: Add `--no-git-push` flag to `main.py`**

In `src/lovebirds/cli/main.py`, after the `--dry-run` argument (line 109), add:

```python
    root.add_argument(
        "--no-git-push",
        dest="no_git_push",
        default=False,
        action="store_true",
        help="Don't automatically commit and push the database file after saving.",
    )
```

Also add `subcommand` defaults to each `set_defaults` call:

```python
    list_cmd.set_defaults(function=list_people, subcommand="list")
    phase_add.set_defaults(function=add_phase, subcommand="phase add")
    reformat.set_defaults(function=reformat_people, subcommand="reformat")
    register.set_defaults(function=import_registrations, subcommand="register")
    resolve.set_defaults(function=resolve_refs, subcommand="resolve-refs")
    send.set_defaults(function=send_messages, subcommand="send")
```

**Step 4: Continue to Task 2** (test still fails — `_git_commit_and_push` doesn't exist yet)

No commit yet — Task 2 completes this feature.

---

### Task 2: Implement `_git_commit_and_push()` in `Config`

**Files:**
- Modify: `src/lovebirds/cli/config.py`
- Test: `tests/cli/test_config.py`

**Step 1: Write the failing tests**

Add to `tests/cli/test_config.py`, continuing the `TestConfigGitPush` class:

```python
import subprocess

class TestConfigGitPush:
    @patch("lovebirds.cli.config.save_people")
    @patch("lovebirds.cli.config.check_type")
    def test_no_git_push_flag_skips_git(self, mock_check, mock_save):
        args = argparse.Namespace(
            dry_run=False,
            people_file="/tmp/test_people.yaml",
            no_git_push=True,
            subcommand="reformat",
        )
        config = Config(args=args, event=None, event_id="", people={})
        with patch.object(config, "_git_commit_and_push") as mock_git:
            config.save_people(backup=False)
        mock_git.assert_not_called()

    @patch("lovebirds.cli.config.save_people")
    @patch("lovebirds.cli.config.check_type")
    def test_git_push_called_after_save(self, mock_check, mock_save):
        args = argparse.Namespace(
            dry_run=False,
            people_file="/tmp/test_people.yaml",
            no_git_push=False,
            subcommand="reformat",
        )
        config = Config(args=args, event=None, event_id="", people={})
        with patch.object(config, "_git_commit_and_push") as mock_git:
            config.save_people(backup=False)
        mock_git.assert_called_once()

    @patch("lovebirds.cli.config.save_people")
    @patch("lovebirds.cli.config.check_type")
    def test_dry_run_skips_git(self, mock_check, mock_save):
        args = argparse.Namespace(
            dry_run=True,
            people_file="/tmp/test_people.yaml",
            no_git_push=False,
            subcommand="reformat",
        )
        config = Config(args=args, event=None, event_id="", people={})
        with patch.object(config, "_git_commit_and_push") as mock_git:
            config.save_people(backup=False)
        mock_git.assert_not_called()


class TestGitCommitAndPush:
    def _make_config(self, people_file: str, subcommand: str = "reformat", event_id: str = "") -> Config:
        args = argparse.Namespace(
            dry_run=False,
            people_file=people_file,
            no_git_push=False,
            subcommand=subcommand,
        )
        return Config(args=args, event=None, event_id=event_id, people={})

    @patch("lovebirds.cli.config.subprocess.run")
    def test_not_in_git_repo_skips(self, mock_run):
        mock_run.side_effect = subprocess.CalledProcessError(128, "git")
        config = self._make_config("/tmp/test.yaml")
        config._git_commit_and_push()
        # Only the rev-parse check was called
        assert mock_run.call_count == 1

    @patch("lovebirds.cli.config.subprocess.run")
    def test_commits_and_pushes(self, mock_run):
        mock_run.return_value = subprocess.CompletedProcess(args=[], returncode=0)
        config = self._make_config("/tmp/test.yaml", subcommand="phase add", event_id="summer2026")
        config._git_commit_and_push()
        calls = mock_run.call_args_list
        # 1: rev-parse, 2: git add, 3: git commit, 4: git push
        assert len(calls) == 4
        assert "--is-inside-work-tree" in calls[0][0][0]
        assert "add" in calls[1][0][0]
        assert "/tmp/test.yaml" in calls[1][0][0]
        commit_cmd = calls[2][0][0]
        assert "commit" in commit_cmd
        assert "/tmp/test.yaml" in commit_cmd
        assert "lovebird: phase add (event: summer2026)" in commit_cmd
        assert "push" in calls[3][0][0]

    @patch("lovebirds.cli.config.subprocess.run")
    def test_commit_message_without_event(self, mock_run):
        mock_run.return_value = subprocess.CompletedProcess(args=[], returncode=0)
        config = self._make_config("/tmp/test.yaml", subcommand="reformat", event_id="")
        config._git_commit_and_push()
        commit_cmd = mock_run.call_args_list[2][0][0]
        assert "lovebird: reformat" in commit_cmd

    @patch("lovebirds.cli.config.subprocess.run")
    def test_nothing_to_commit_skips_push(self, mock_run):
        def side_effect(*args, **kwargs):
            cmd = args[0]
            if "commit" in cmd:
                raise subprocess.CalledProcessError(1, "git")
            return subprocess.CompletedProcess(args=[], returncode=0)
        mock_run.side_effect = side_effect
        config = self._make_config("/tmp/test.yaml")
        config._git_commit_and_push()
        push_calls = [c for c in mock_run.call_args_list if "push" in c[0][0]]
        assert len(push_calls) == 0
```

**Step 2: Run tests to verify they fail**

Run: `poetry run pytest tests/cli/test_config.py::TestConfigGitPush tests/cli/test_config.py::TestGitCommitAndPush -v`
Expected: FAIL — `_git_commit_and_push` does not exist

**Step 3: Write implementation**

In `src/lovebirds/cli/config.py`, add `import subprocess` to the imports, then add the method and modify `save_people`:

```python
import subprocess

@dataclass(kw_only=True)
class Config:
    args: argparse.Namespace
    event: Event | None
    event_id: EventId
    people: People
    event_participants_stats: EventParticipantsStatistics | None = None

    def save_people(self, backup: bool = True) -> None:
        try:
            check_type(self.people, People)
        except TypeCheckError as exc:
            logging.error(f"Type checking of people failed: {exc}.")
            if not self.args.dry_run:
                logging.error("Continuing may be dangerous.")
                input("Hit <Enter> to continue anyway...")

        if self.args.dry_run:
            return

        if backup:
            backup_file(self.args.people_file)

        # Saving is critical: loop until we succeed.
        while True:
            try:
                save_people(self.args.people_file, self.people)
                break
            except BaseException as exc:
                traceback.print_exc()
                input("Database saving failed. Press enter to retry.")

        if not self.args.no_git_push:
            self._git_commit_and_push()

    def _git_commit_and_push(self) -> None:
        people_file = self.args.people_file
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

        # Build commit message
        subcommand = self.args.subcommand
        if self.event_id:
            message = f"lovebird: {subcommand} (event: {self.event_id})"
        else:
            message = f"lovebird: {subcommand}"

        # Stage and commit only the database file
        subprocess.run(
            ["git", "-C", repo_dir, "add", people_file],
            check=True,
        )
        try:
            subprocess.run(
                ["git", "-C", repo_dir, "commit", people_file, "-m", message],
                check=True,
            )
        except subprocess.CalledProcessError:
            logging.info("Nothing to commit.")
            return

        # Push with interactive retry
        while True:
            try:
                subprocess.run(
                    ["git", "-C", repo_dir, "push"],
                    check=True,
                )
                break
            except subprocess.CalledProcessError:
                traceback.print_exc()
                input("Git push failed. Press enter to retry.")
```

Also add `import os` to the imports in `config.py`.

**Step 4: Run tests to verify they pass**

Run: `poetry run pytest tests/cli/test_config.py -v`
Expected: All PASS

**Step 5: Commit**

```bash
git add src/lovebirds/cli/config.py src/lovebirds/cli/main.py tests/cli/test_config.py
git commit -m "feat: auto git commit and push database after save"
```

---

### Task 3: Type checking and formatting

**Step 1: Run mypy**

Run: `poetry run mypy src/`
Expected: No errors.

**Step 2: Run black**

Run: `poetry run black src/ tests/`
Expected: Reformatted if needed.

**Step 3: Run full test suite**

Run: `poetry run pytest`
Expected: All tests pass.

**Step 4: Commit if anything changed**

```bash
git add -u
git commit -m "chore: fix formatting and type issues"
```

---

### Task 4: Update existing test helper

The existing `_make_config` helper in `tests/cli/test_config.py` needs `no_git_push` and `subcommand` added to the `Namespace` so existing tests don't break when `save_people` tries to access `args.no_git_push`.

**Step 1: Check if existing tests pass**

Run: `poetry run pytest tests/cli/test_config.py -v`

If the existing `TestConfigSavePeople` tests fail because `args` is missing `no_git_push` or `subcommand`, update `_make_config`:

```python
def _make_config(dry_run: bool = False, people: People | None = None) -> Config:
    if people is None:
        people = {}
    args = argparse.Namespace(
        dry_run=dry_run,
        people_file="/tmp/test_people.yaml",
        no_git_push=True,
        subcommand="test",
    )
    return Config(
        args=args,
        event=None,
        event_id="",
        people=people,
    )
```

Note: `no_git_push=True` in the helper so existing tests don't trigger git operations.

**Step 2: Run tests**

Run: `poetry run pytest tests/cli/test_config.py -v`
Expected: All pass.

**Step 3: Commit if changed**

```bash
git add tests/cli/test_config.py
git commit -m "chore: update test helper with git push defaults"
```
