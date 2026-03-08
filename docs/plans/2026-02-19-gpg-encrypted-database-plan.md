# GPG-Encrypted Database — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Transparently decrypt/encrypt the people database file when it has a `.gpg` extension, reading recipient key IDs from a `.gpg-id` file.

**Architecture:** All GPG logic lives in `src/lovebirds/io.py`. Four new internal functions (`_is_gpg_file`, `_read_gpg_ids`, `_gpg_decrypt`, `_gpg_encrypt`) are added, and `load_people`/`save_people` gain a `.gpg` extension check to route through them. No other modules change.

**Tech Stack:** Python 3.13+, subprocess (gpg CLI), pytest + pytest-mock for testing.

**Design doc:** `docs/plans/2026-02-19-gpg-encrypted-database-design.md`

---

### Task 1: `_is_gpg_file` — extension detection helper

**Files:**
- Modify: `src/lovebirds/io.py`
- Test: `tests/test_io.py`

**Step 1: Write the failing tests**

Add to `tests/test_io.py`:

```python
from lovebirds.io import _is_gpg_file

class TestIsGpgFile:
    def test_gpg_extension(self):
        assert _is_gpg_file("people.yaml.gpg") is True

    def test_yaml_extension(self):
        assert _is_gpg_file("people.yaml") is False

    def test_pathlike(self, tmp_path):
        assert _is_gpg_file(tmp_path / "db.yaml.gpg") is True

    def test_no_extension(self):
        assert _is_gpg_file("people") is False
```

**Step 2: Run tests to verify they fail**

Run: `poetry run pytest tests/test_io.py::TestIsGpgFile -v`
Expected: FAIL — `ImportError: cannot import name '_is_gpg_file'`

**Step 3: Write minimal implementation**

Add to `src/lovebirds/io.py`, after the existing imports:

```python
def _is_gpg_file(filename: FilePath) -> bool:
    return str(filename).endswith(".gpg")
```

**Step 4: Run tests to verify they pass**

Run: `poetry run pytest tests/test_io.py::TestIsGpgFile -v`
Expected: 4 PASS

**Step 5: Commit**

```bash
git add src/lovebirds/io.py tests/test_io.py
git commit -m "feat: add _is_gpg_file extension detection helper"
```

---

### Task 2: `_read_gpg_ids` — parse `.gpg-id` file

**Files:**
- Modify: `src/lovebirds/io.py`
- Test: `tests/test_io.py`

**Step 1: Write the failing tests**

Add to `tests/test_io.py`:

```python
from lovebirds.io import _read_gpg_ids

class TestReadGpgIds:
    def test_single_id(self, tmp_path):
        gpg_id_file = tmp_path / ".gpg-id"
        gpg_id_file.write_text("ABCD1234\n")
        db_path = tmp_path / "people.yaml.gpg"
        assert _read_gpg_ids(db_path) == ["ABCD1234"]

    def test_multiple_ids(self, tmp_path):
        gpg_id_file = tmp_path / ".gpg-id"
        gpg_id_file.write_text("ABCD1234\nEFGH5678\n")
        db_path = tmp_path / "people.yaml.gpg"
        assert _read_gpg_ids(db_path) == ["ABCD1234", "EFGH5678"]

    def test_ignores_comments_and_blank_lines(self, tmp_path):
        gpg_id_file = tmp_path / ".gpg-id"
        gpg_id_file.write_text("# A comment\n\nABCD1234\n  \n# Another\nEFGH5678\n")
        db_path = tmp_path / "people.yaml.gpg"
        assert _read_gpg_ids(db_path) == ["ABCD1234", "EFGH5678"]

    def test_strips_whitespace(self, tmp_path):
        gpg_id_file = tmp_path / ".gpg-id"
        gpg_id_file.write_text("  ABCD1234  \n")
        db_path = tmp_path / "people.yaml.gpg"
        assert _read_gpg_ids(db_path) == ["ABCD1234"]

    def test_missing_file_raises(self, tmp_path):
        db_path = tmp_path / "people.yaml.gpg"
        with pytest.raises(FileNotFoundError, match=".gpg-id"):
            _read_gpg_ids(db_path)

    def test_empty_file_raises(self, tmp_path):
        gpg_id_file = tmp_path / ".gpg-id"
        gpg_id_file.write_text("# only comments\n\n")
        db_path = tmp_path / "people.yaml.gpg"
        with pytest.raises(ValueError, match="No GPG key IDs"):
            _read_gpg_ids(db_path)
```

**Step 2: Run tests to verify they fail**

Run: `poetry run pytest tests/test_io.py::TestReadGpgIds -v`
Expected: FAIL — `ImportError: cannot import name '_read_gpg_ids'`

**Step 3: Write minimal implementation**

Add to `src/lovebirds/io.py`:

```python
def _read_gpg_ids(database_path: FilePath) -> list[str]:
    gpg_id_path = os.path.join(os.path.dirname(str(database_path)), ".gpg-id")
    if not os.path.isfile(gpg_id_path):
        raise FileNotFoundError(
            f"No .gpg-id file found in {os.path.dirname(str(database_path))}. "
            f"Required for encrypting {database_path}."
        )
    with open(gpg_id_path, "r", encoding="utf-8") as f:
        ids = [
            line.strip()
            for line in f
            if line.strip() and not line.strip().startswith("#")
        ]
    if not ids:
        raise ValueError(f"No GPG key IDs found in {gpg_id_path}")
    return ids
```

**Step 4: Run tests to verify they pass**

Run: `poetry run pytest tests/test_io.py::TestReadGpgIds -v`
Expected: 6 PASS

**Step 5: Commit**

```bash
git add src/lovebirds/io.py tests/test_io.py
git commit -m "feat: add _read_gpg_ids to parse .gpg-id file"
```

---

### Task 3: `_gpg_decrypt` — decrypt a GPG file

**Files:**
- Modify: `src/lovebirds/io.py` (add `import subprocess` and the function)
- Test: `tests/test_io.py`

**Step 1: Write the failing tests**

Add to `tests/test_io.py`:

```python
import subprocess
from unittest.mock import patch

from lovebirds.io import _gpg_decrypt

class TestGpgDecrypt:
    def test_calls_gpg_decrypt(self, tmp_path):
        gpg_file = tmp_path / "data.gpg"
        gpg_file.write_bytes(b"encrypted")
        with patch("lovebirds.io.subprocess.run") as mock_run:
            mock_run.return_value = subprocess.CompletedProcess(
                args=[], returncode=0, stdout=b"decrypted yaml"
            )
            result = _gpg_decrypt(gpg_file)
        mock_run.assert_called_once_with(
            ["gpg", "--decrypt", str(gpg_file)],
            stdout=subprocess.PIPE,
            check=True,
            text=False,
        )
        assert result == b"decrypted yaml"

    def test_gpg_failure_raises(self, tmp_path):
        gpg_file = tmp_path / "data.gpg"
        gpg_file.write_bytes(b"encrypted")
        with patch("lovebirds.io.subprocess.run") as mock_run:
            mock_run.side_effect = subprocess.CalledProcessError(2, "gpg")
            with pytest.raises(subprocess.CalledProcessError):
                _gpg_decrypt(gpg_file)
```

**Step 2: Run tests to verify they fail**

Run: `poetry run pytest tests/test_io.py::TestGpgDecrypt -v`
Expected: FAIL — `ImportError: cannot import name '_gpg_decrypt'`

**Step 3: Write minimal implementation**

Add `import subprocess` to the imports in `src/lovebirds/io.py`, then add:

```python
def _gpg_decrypt(filename: FilePath) -> bytes:
    result = subprocess.run(
        ["gpg", "--decrypt", str(filename)],
        stdout=subprocess.PIPE,
        check=True,
        text=False,
    )
    return result.stdout
```

**Step 4: Run tests to verify they pass**

Run: `poetry run pytest tests/test_io.py::TestGpgDecrypt -v`
Expected: 2 PASS

**Step 5: Commit**

```bash
git add src/lovebirds/io.py tests/test_io.py
git commit -m "feat: add _gpg_decrypt subprocess wrapper"
```

---

### Task 4: `_gpg_encrypt` — encrypt data for recipients

**Files:**
- Modify: `src/lovebirds/io.py`
- Test: `tests/test_io.py`

**Step 1: Write the failing tests**

Add to `tests/test_io.py`:

```python
from lovebirds.io import _gpg_encrypt

class TestGpgEncrypt:
    def test_calls_gpg_encrypt_single_recipient(self):
        with patch("lovebirds.io.subprocess.run") as mock_run:
            mock_run.return_value = subprocess.CompletedProcess(
                args=[], returncode=0, stdout=b"encrypted data"
            )
            result = _gpg_encrypt(b"plain yaml", ["ABCD1234"])
        mock_run.assert_called_once_with(
            ["gpg", "--encrypt", "--recipient", "ABCD1234"],
            input=b"plain yaml",
            stdout=subprocess.PIPE,
            check=True,
        )
        assert result == b"encrypted data"

    def test_calls_gpg_encrypt_multiple_recipients(self):
        with patch("lovebirds.io.subprocess.run") as mock_run:
            mock_run.return_value = subprocess.CompletedProcess(
                args=[], returncode=0, stdout=b"encrypted data"
            )
            result = _gpg_encrypt(b"plain yaml", ["ABCD1234", "EFGH5678"])
        expected_cmd = [
            "gpg", "--encrypt",
            "--recipient", "ABCD1234",
            "--recipient", "EFGH5678",
        ]
        mock_run.assert_called_once_with(
            expected_cmd,
            input=b"plain yaml",
            stdout=subprocess.PIPE,
            check=True,
        )
        assert result == b"encrypted data"

    def test_gpg_failure_raises(self):
        with patch("lovebirds.io.subprocess.run") as mock_run:
            mock_run.side_effect = subprocess.CalledProcessError(2, "gpg")
            with pytest.raises(subprocess.CalledProcessError):
                _gpg_encrypt(b"plain yaml", ["ABCD1234"])
```

**Step 2: Run tests to verify they fail**

Run: `poetry run pytest tests/test_io.py::TestGpgEncrypt -v`
Expected: FAIL — `ImportError: cannot import name '_gpg_encrypt'`

**Step 3: Write minimal implementation**

Add to `src/lovebirds/io.py`:

```python
def _gpg_encrypt(data: bytes, recipient_ids: list[str]) -> bytes:
    cmd = ["gpg", "--encrypt"]
    for rid in recipient_ids:
        cmd.extend(["--recipient", rid])
    result = subprocess.run(
        cmd,
        input=data,
        stdout=subprocess.PIPE,
        check=True,
    )
    return result.stdout
```

**Step 4: Run tests to verify they pass**

Run: `poetry run pytest tests/test_io.py::TestGpgEncrypt -v`
Expected: 3 PASS

**Step 5: Commit**

```bash
git add src/lovebirds/io.py tests/test_io.py
git commit -m "feat: add _gpg_encrypt subprocess wrapper"
```

---

### Task 5: Wire GPG into `load_people`

**Files:**
- Modify: `src/lovebirds/io.py` (modify `load_people`)
- Test: `tests/test_io.py`

**Step 1: Write the failing tests**

Add to `tests/test_io.py`:

```python
class TestLoadPeopleGpg:
    def test_load_gpg_file_decrypts(self, tmp_path, sample_people):
        # First, save plain YAML so we know what the decrypted content looks like
        plain_path = tmp_path / "people.yaml"
        save_people(plain_path, sample_people)
        yaml_bytes = plain_path.read_bytes()

        gpg_path = tmp_path / "people.yaml.gpg"
        gpg_path.write_bytes(b"encrypted")

        with patch("lovebirds.io._gpg_decrypt") as mock_decrypt:
            mock_decrypt.return_value = yaml_bytes
            loaded = load_people(gpg_path)

        mock_decrypt.assert_called_once_with(gpg_path)
        assert set(loaded.keys()) == set(sample_people.keys())

    def test_load_plain_file_unchanged(self, tmp_path, sample_people):
        plain_path = tmp_path / "people.yaml"
        save_people(plain_path, sample_people)

        with patch("lovebirds.io._gpg_decrypt") as mock_decrypt:
            loaded = load_people(plain_path)

        mock_decrypt.assert_not_called()
        assert set(loaded.keys()) == set(sample_people.keys())
```

**Step 2: Run tests to verify they fail**

Run: `poetry run pytest tests/test_io.py::TestLoadPeopleGpg -v`
Expected: FAIL — `test_load_gpg_file_decrypts` fails because `load_people` tries to `open()` a `.gpg` file as YAML

**Step 3: Modify `load_people`**

In `src/lovebirds/io.py`, change `load_people` from:

```python
def load_people(filename: FilePath) -> p.People:
    with open(filename, "r", encoding="utf-8") as file:
        return _people_decoder.decode(file.read())
```

to:

```python
def load_people(filename: FilePath) -> p.People:
    if _is_gpg_file(filename):
        data = _gpg_decrypt(filename)
        return _people_decoder.decode(data.decode("utf-8"))
    with open(filename, "r", encoding="utf-8") as file:
        return _people_decoder.decode(file.read())
```

**Step 4: Run tests to verify they pass**

Run: `poetry run pytest tests/test_io.py::TestLoadPeopleGpg tests/test_io.py::TestSaveLoadRoundTrip -v`
Expected: All PASS (new GPG tests pass, existing round-trip still passes)

**Step 5: Commit**

```bash
git add src/lovebirds/io.py tests/test_io.py
git commit -m "feat: load_people transparently decrypts .gpg files"
```

---

### Task 6: Wire GPG into `save_people`

**Files:**
- Modify: `src/lovebirds/io.py` (modify `save_people`)
- Test: `tests/test_io.py`

**Step 1: Write the failing tests**

Add to `tests/test_io.py`:

```python
from lovebirds.io import _read_gpg_ids

class TestSavePeopleGpg:
    def test_save_gpg_file_encrypts(self, tmp_path, sample_people):
        gpg_id_file = tmp_path / ".gpg-id"
        gpg_id_file.write_text("ABCD1234\nEFGH5678\n")
        gpg_path = tmp_path / "people.yaml.gpg"

        with patch("lovebirds.io._gpg_encrypt") as mock_encrypt:
            mock_encrypt.return_value = b"encrypted output"
            save_people(gpg_path, sample_people)

        mock_encrypt.assert_called_once()
        args = mock_encrypt.call_args
        # First arg is YAML bytes, second is recipient list
        assert isinstance(args[0][0], bytes)
        assert args[0][1] == ["ABCD1234", "EFGH5678"]
        # The file should contain the encrypted output
        assert gpg_path.read_bytes() == b"encrypted output"

    def test_save_plain_file_unchanged(self, tmp_path, sample_people):
        plain_path = tmp_path / "people.yaml"

        with patch("lovebirds.io._gpg_encrypt") as mock_encrypt:
            save_people(plain_path, sample_people)

        mock_encrypt.assert_not_called()
        # File should be readable YAML
        loaded = load_people(plain_path)
        assert set(loaded.keys()) == set(sample_people.keys())

    def test_save_gpg_missing_gpg_id_raises(self, tmp_path, sample_people):
        gpg_path = tmp_path / "people.yaml.gpg"
        with pytest.raises(FileNotFoundError, match=".gpg-id"):
            save_people(gpg_path, sample_people)
```

**Step 2: Run tests to verify they fail**

Run: `poetry run pytest tests/test_io.py::TestSavePeopleGpg -v`
Expected: FAIL — `save_people` writes plain YAML regardless of extension

**Step 3: Modify `save_people`**

In `src/lovebirds/io.py`, change `save_people` from:

```python
def save_people(filename: FilePath, people: p.People) -> None:
    sorted_people = p.sorted_people(people)
    dict_data = _people_encoder.encode(sorted_people)
    yaml_data = yaml.safe_dump(
        dict_data,
        stream=None,
        allow_unicode=True,
        default_flow_style=None,
        encoding="utf-8",
        indent=2,
        sort_keys=False,
        width=512,
    )
    _safe_write_file(filename, yaml_data)
```

to:

```python
def save_people(filename: FilePath, people: p.People) -> None:
    sorted_people = p.sorted_people(people)
    dict_data = _people_encoder.encode(sorted_people)
    yaml_data = yaml.safe_dump(
        dict_data,
        stream=None,
        allow_unicode=True,
        default_flow_style=None,
        encoding="utf-8",
        indent=2,
        sort_keys=False,
        width=512,
    )
    if _is_gpg_file(filename):
        recipient_ids = _read_gpg_ids(filename)
        encrypted_data = _gpg_encrypt(yaml_data, recipient_ids)
        _safe_write_file(filename, encrypted_data)
    else:
        _safe_write_file(filename, yaml_data)
```

**Step 4: Run tests to verify they pass**

Run: `poetry run pytest tests/test_io.py -v`
Expected: All PASS (new GPG tests + all existing tests)

**Step 5: Commit**

```bash
git add src/lovebirds/io.py tests/test_io.py
git commit -m "feat: save_people transparently encrypts .gpg files"
```

---

### Task 7: Type checking and formatting

**Files:**
- Check: `src/lovebirds/io.py`, `tests/test_io.py`

**Step 1: Run mypy**

Run: `poetry run mypy src/`
Expected: No errors. If there are type errors in the new code, fix them.

**Step 2: Run black**

Run: `poetry run black src/ tests/`
Expected: Files reformatted if needed.

**Step 3: Run full test suite**

Run: `poetry run pytest`
Expected: All tests pass.

**Step 4: Commit if anything changed**

```bash
git add -u
git commit -m "chore: fix formatting and type issues"
```

(Skip this commit if nothing changed.)

---

### Task 8: Manual integration smoke test

**Step 1: Create a test GPG key (if none exists)**

```bash
gpg --batch --gen-key <<EOF
%no-protection
Key-Type: RSA
Key-Length: 2048
Name-Real: Test User
Name-Email: test@example.com
Expire-Date: 0
EOF
```

**Step 2: Create a `.gpg-id` file and encrypted database**

```bash
mkdir -p /tmp/lovebirds-test
# Get the key ID
KEY_ID=$(gpg --list-keys --keyid-format long test@example.com | grep -oP '(?<=/)[A-F0-9]{16}')
echo "$KEY_ID" > /tmp/lovebirds-test/.gpg-id

# Create a minimal YAML database and encrypt it
cat > /tmp/lovebirds-test/people.yaml <<YAMLEOF
test@example.com:
  email: test@example.com
  languages: [en]
YAMLEOF
gpg --encrypt --recipient "$KEY_ID" --output /tmp/lovebirds-test/people.yaml.gpg /tmp/lovebirds-test/people.yaml
rm /tmp/lovebirds-test/people.yaml
```

**Step 3: Test load + reformat (round-trip)**

Run: `poetry run lovebird -p /tmp/lovebirds-test/people.yaml.gpg reformat`
Expected: Command completes without error. A backup `.gpg` file is created. The rewritten `.gpg` file can be decrypted:

```bash
gpg --decrypt /tmp/lovebirds-test/people.yaml.gpg
```

Should output valid YAML with the test person.

**Step 4: Clean up**

```bash
rm -rf /tmp/lovebirds-test
```
