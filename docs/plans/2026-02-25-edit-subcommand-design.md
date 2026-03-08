# Edit Subcommand Design

## Overview

A new `edit` subcommand that opens the entire people database in `$EDITOR` for manual editing. If the database is GPG-encrypted, it is decrypted before editing and re-encrypted after saving. Invalid YAML or schema errors re-open the editor so the user can fix them.

## Flow

1. `main.py` loads the database via `load_people()` (transparently decrypts `.gpg` files)
2. `edit_people()` calls `edit_as_yaml(People, People, config.people, prefix)` — dumps to temp file, opens `$EDITOR`, validates on close, loops on error
3. On success, assigns result to `config.people` and calls `config.save_people()` — backup, type check, re-encrypt if `.gpg`, git auto-commit

## Subcommand registration

Top-level subcommand alongside `list`, `reformat`, etc. No extra arguments.

```
lovebird -p people.yaml.gpg edit
```

`set_defaults(function=edit_people, subcommand="edit")` produces git commit messages like `lovebird: edit`.

## Changes

- **New:** `src/lovebirds/cli/edit_cmd.py` — `edit_people(config)` function
- **Modified:** `src/lovebirds/cli/main.py` — import and register the subcommand
- **Unchanged:** `io.py`, `config.py`, `edit.py`
