# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Lovebirds is a Python CLI tool and library for managing event participation data and sending messages for The Lovers' Guild. It operates on a human-editable YAML database of participants.

## Build & Development Commands

```sh
poetry install                          # Install dependencies into virtualenv
poetry run lovebird <subcommand>        # Run the CLI tool
poetry run black src/                   # Format code
poetry run mypy src/                    # Type check (strict mode, see mypy.ini)
poetry run deadcode src/                # Find unused code
```

No test suite exists yet.

## Architecture

The CLI entry point is `lovebird` → `lovebirds.cli.main:main`. It uses argparse with hierarchical subcommands: `list`, `phase add`, `register`, `send`, `reformat`, `resolve-refs`. Every subcommand requires `-p <people.yaml>` pointing to the participant database file.

### Key layers

- **`models/`** — Strongly-typed dataclasses with Mashumaro (de)serialization. `Person` is the core record; `Participation` tracks per-event status via append-only `ParticipationPhase` entries. Custom `EmailAddress` type with validation. A typeguard checker plugin is registered via the `typeguard.checker_lookup` entry point in pyproject.toml.

- **`io.py`** — YAML-based database I/O. Writes are atomic (temp file + rename) with automatic timestamped backups. Uses a custom YAML representer for human-readable output (flow-style for scalar lists).

- **`cli/`** — Each subcommand in its own module. `config.py` holds the shared `Config` dataclass passed to all subcommand functions. `edit.py` provides interactive editing (readline + external `$EDITOR`). `sendmail.py` handles SMTP with TLS, message tracking to prevent duplicate sends, and Maildir/mbox archiving.

- **`templates/`** — Jinja2-based template evaluation with recursive variable resolution and forward references. `person.py` converts `Person` objects to template-safe dicts. Templates are used in the `list`, `phase`, and `send` subcommands for filtering, column expressions, and message generation.

- **`statistics.py`** — Computes participant statistics (counts, age ranges, averages) per event, exposed as a `ParticipantsStatistics` dataclass available in templates.

### External tool dependencies

- **GPG** — Decrypts registration files during `register`
- **Pandoc** — Converts markdown to HTML for emails (templates in `data/pandoc-templates/`)
- **$EDITOR/$VISUAL** — Used for interactive YAML editing during registration

## Code Conventions

- Python 3.13+ required
- Strict mypy configuration (all `disallow_*` and `warn_*` flags enabled)
- Black for formatting
- GPL-3.0 license header on every source file (see `py.template` for the format)
- All database writes must be atomic with backup creation
- Participation phases are append-only (immutable history)
