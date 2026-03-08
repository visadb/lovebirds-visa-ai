# GPG-Encrypted Database Support

## Overview

Add transparent GPG encryption/decryption for the people database file. When the database file has a `.gpg` extension, the tool automatically decrypts it on load and re-encrypts it on save. Recipient key IDs are read from a `.gpg-id` file in the same directory, matching the convention used by the `pass` password manager.

## Detection

- A people file is GPG-encrypted if its path ends with `.gpg` (e.g. `people.yaml.gpg`)
- Plain `.yaml` files continue to work as before, unchanged
- When a `.gpg` file is detected, the tool reads `.gpg-id` from the same directory as the database file
- `.gpg-id` format: one GPG key ID per line, blank lines and `#` comments ignored
- If `.gpg-id` is missing or empty when a `.gpg` file is used, the tool raises a clear error

## Decryption (load path)

- `load_people()` checks if `filename` ends with `.gpg`
- If yes: runs `gpg --decrypt <filename>` via subprocess, captures decrypted YAML bytes, then decodes as normal
- If no: reads the file as plain YAML (current behavior, unchanged)
- GPG errors (bad passphrase, missing key) propagate as `subprocess.CalledProcessError`

## Encryption (save path)

- `save_people()` checks if `filename` ends with `.gpg`
- If yes:
  1. Back up the existing `.gpg` file as-is (encrypted copy via `backup_file()`)
  2. Serialize people to YAML bytes (same as current)
  3. Read `.gpg-id` to get recipient key IDs
  4. Run `gpg --encrypt --recipient <id1> --recipient <id2> ...` with YAML piped via stdin, capture encrypted output
  5. Write encrypted output atomically (temp file + rename via `_safe_write_file`)
- If no: current plain YAML write path, unchanged

## `.gpg-id` parsing

A new internal function `_read_gpg_ids(database_path)`:

- Derives `.gpg-id` path as `os.path.join(os.path.dirname(database_path), ".gpg-id")`
- Reads the file, strips whitespace, ignores blank lines and lines starting with `#`
- Returns a `list[str]` of key IDs
- Raises `FileNotFoundError` if `.gpg-id` doesn't exist
- Raises `ValueError` if the file contains no key IDs

## Error handling

- **Missing `.gpg-id`**: `FileNotFoundError` with message like `"No .gpg-id file found in <directory>. Required for encrypting <filename>."`
- **Empty `.gpg-id`**: `ValueError` with `"No GPG key IDs found in <path>/.gpg-id"`
- **GPG decryption/encryption failure**: `subprocess.CalledProcessError` propagates naturally with gpg's stderr
- **No retry logic**: if GPG fails, the tool exits. The original encrypted file is untouched (backup before write, atomic write prevents partial overwrites)

## Changes summary

### Modified in `io.py`

- New internal: `_is_gpg_file(filename)` — checks `.gpg` extension
- New internal: `_read_gpg_ids(database_path)` — parses `.gpg-id`
- New internal: `_gpg_decrypt(filename)` — runs `gpg --decrypt`, returns decrypted bytes
- New internal: `_gpg_encrypt(data, recipient_ids)` — runs `gpg --encrypt`, returns encrypted bytes
- Modified: `load_people()` — detects `.gpg`, decrypts before YAML decode
- Modified: `save_people()` — detects `.gpg`, encrypts after YAML serialize

### No changes needed

- `backup_file()` — copies files as-is, works for `.gpg` files unchanged
- `main.py` — passes `args.people_file` through, unaware of encryption
- `config.py` — `save_people()` call uses the original filename
- All subcommand modules — completely unaware of encryption
- `load_event()` — event files are not encrypted
