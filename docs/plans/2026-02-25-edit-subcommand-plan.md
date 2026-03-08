# Edit Subcommand — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add an `edit` subcommand that opens the people database in `$EDITOR` with validation and GPG-transparent save.

**Architecture:** New `edit_cmd.py` module with a single handler that calls the existing `edit_as_yaml()` utility, then `config.save_people()`. Registered in `main.py` like every other subcommand.

**Tech Stack:** Python 3.13+, pytest + unittest.mock for testing.

**Design doc:** `docs/plans/2026-02-25-edit-subcommand-design.md`

---

### Task 1: Create `edit_cmd.py` and register the subcommand

**Files:**
- Create: `src/lovebirds/cli/edit_cmd.py`
- Modify: `src/lovebirds/cli/main.py`
- Test: `tests/cli/test_edit_cmd.py`

**Step 1: Write the failing test**

Create `tests/cli/test_edit_cmd.py`:

```python
import argparse
from unittest.mock import patch

from lovebirds.cli.config import Config
from lovebirds.cli.edit_cmd import edit_people
from lovebirds.models.people import People


class TestEditPeople:
    @patch("lovebirds.cli.edit_cmd.edit_as_yaml")
    def test_calls_edit_as_yaml_and_saves(self, mock_edit):
        people: People = {}
        edited: People = {}
        mock_edit.return_value = edited

        args = argparse.Namespace(
            dry_run=False,
            people_file="/tmp/test.yaml",
            no_git_push=True,
            subcommand="edit",
        )
        config = Config(args=args, event=None, event_id="", people=people)

        with patch.object(config, "save_people") as mock_save:
            edit_people(config)

        mock_edit.assert_called_once_with(People, People, people, edit_people.EDIT_PREFIX)
        assert config.people is edited
        mock_save.assert_called_once()
```

**Step 2: Run test to verify it fails**

Run: `poetry run pytest tests/cli/test_edit_cmd.py -v`
Expected: FAIL — `edit_cmd` module does not exist

**Step 3: Write implementation**

Create `src/lovebirds/cli/edit_cmd.py`:

```python
# ©2025 The Lovers' Guild
# This file is licensed under the GNU General Public License version 3.0.

"""Edit participation database in $EDITOR."""

from lovebirds.cli.config import Config
from lovebirds.cli.edit import edit_as_yaml
from lovebirds.models.people import People


def edit_people(config: Config) -> None:
    config.people = edit_as_yaml(
        People, People, config.people, edit_people.EDIT_PREFIX
    )
    config.save_people()


edit_people.EDIT_PREFIX = "Editing people database. Save and close editor when done."
```

Note: attaching the prefix as a function attribute keeps it accessible for testing without a module-level constant.

**Step 4: Register in `main.py`**

Add import at line 12 (with the other subcommand imports):

```python
from lovebirds.cli.edit_cmd import edit_people
```

After the `reformat` block (after line 223), add:

```python
    edit = root_sub.add_parser(
        "edit",
        help="Open participation database in $EDITOR for manual editing.",
    )
    edit.set_defaults(function=edit_people, subcommand="edit")
```

**Step 5: Run tests to verify they pass**

Run: `poetry run pytest tests/cli/test_edit_cmd.py -v`
Expected: PASS

**Step 6: Run full test suite**

Run: `poetry run pytest -q`
Expected: All pass

**Step 7: Run mypy and black**

Run: `poetry run mypy src/ && poetry run black src/ tests/`
Expected: Clean

**Step 8: Commit**

```bash
git add src/lovebirds/cli/edit_cmd.py src/lovebirds/cli/main.py tests/cli/test_edit_cmd.py
git commit -m "feat: add edit subcommand for manual database editing"
```
