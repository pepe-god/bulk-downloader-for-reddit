# Plan: Python 3.14 upgrade, dependency refresh, add ruff, remove MagicMock/

## Goal
- Raise the project's Python floor and uv/local version to **3.14.7 (latest stable)**.
- Refresh all dependencies to their latest compatible releases via `uv lock --upgrade`.
- Add **ruff** as a dev dependency and make it the linter + import sorter (replacing flake8 + isort).
- Delete the stray `MagicMock/` directory.

## Decisions (confirmed with user)
- Python target = **3.14.7 stable** (not 3.15rc). `.python-version` = `3.14`, `requires-python` = `>=3.14`.
- File removal = **only `MagicMock/`** (untracked stray test artifact; verified not tracked, not in `.gitignore`). `tox.ini`, `.pre-commit-config.yaml`, `formatting_check.yml`, `protect_master.yml` are kept.
- ruff replaces flake8 + isort. **black is kept** for formatting (ruff only takes over lint + import sorting).

## Constraints (from AGENTS.md — must preserve)
- **Do NOT bump praw to >=8** — keep `praw>=7.2.0,<8`. praw 8 removes APIs used by `bdfr/oauth2.py`.
- Keep `prawcore>=2.0.0` (used directly in `oauth2.py`).
- `bdfr.__version__` in `bdfr/__init__.py` is dynamic — do not remove.
- Four console scripts stay in `bdfr/__main__.py`.

---

## Steps

### 1. Python version
- Edit `.python-version`: `3.11` → `3.14`.
- Edit `pyproject.toml`:
  - `requires-python = ">=3.14"`
  - Classifiers: drop `3.11`/`3.12`/`3.13` lines; keep `"Programming Language :: Python :: 3"` and add `"Programming Language :: Python :: 3.14"`.
  - `[tool.black] target-version = ["py311"]` → `["py314"]`.
- Install interpreter: `uv python install 3.14` (downloads 3.14.7).
- `.github/workflows/test.yml`: matrix `python-version: ["3.11","3.12","3.13"]` → `["3.14"]`; windows entry → `3.14`.
- `.github/workflows/publish.yml`: `python-version: '3.11'` → `'3.14'`.
- `AGENTS.md`: update "Minimum Python is 3.11 (requires-python = \">=3.11\"); CI tests 3.11/3.12/3.13" → 3.14, and the flake8 lint notes → ruff.

### 2. Dependencies
- `pyproject.toml` `[project.optional-dependencies] dev`:
  - Add `"ruff>=0.8.0"`.
  - Remove `"Flake8-pyproject>=1.2.2"` and `"isort>=5.11.4"` (replaced by ruff). Keep `black`, `pre-commit`, `pytest`, `tox`.
- Refresh + resolve: `uv lock --upgrade` (respects the `praw<8` / `prawcore>=2.0.0` caps). If a dep has no 3.14-compatible release, pin the last working version rather than dropping it.

### 3. ruff config (replace flake8 + isort)
- Remove `[tool.flake8]` and `[tool.isort]` sections from `pyproject.toml`.
- Add:
  ```toml
  [tool.ruff]
  line-length = 120
  target-version = "py314"

  [tool.ruff.lint]
  select = ["E", "F", "I", "W", "UP", "B", "C4", "SIM"]

  [tool.ruff.lint.isort]
  known-first-party = ["bdfr"]
  ```

### 4. Wire ruff into tooling (files kept, content updated)
- `.pre-commit-config.yaml`: drop the `psf/black`? keep black hook; replace `pycqa/isort` and `pycqa/flake8` repos with `astral-sh/ruff-pre-commit` (`ruff check` hook). Remove `Flake8-pyproject` `additional_dependencies`.
- `tox.ini` `format` env: `isort` + `black` → `ruff check --fix` + `black`. `format_check` env: `isort --check` + `black --check` + `mdl` → `ruff check` + `black --check` + `mdl`.
- `.github/workflows/test.yml`:
  - Install step: replace `Flake8-pyproject` with `ruff` (`pip install --upgrade pip ruff pytest pytest-cov`).
  - "Lint with flake8" step (`flake8 . --select=E9,F63,F7,F82`) → `ruff check .` (covers E/F/I; mirrors the prior syntax-critical intent).

### 5. Remove stray directory
- `rm -rf MagicMock` (untracked; no git commit needed for it, but it disappears from the tree).

---

## Validation
Run inside the 3.14 env:
- `uv sync --all-extras`
- `uv run ruff check bdfr tests` — fix safe findings; review/resolve any new ones (do not silently ignore logic errors).
- `uv run black --check .` (formatting must still pass).
- `uv run pytest -m "not online and not reddit and not authenticated" --deselect tests/test_downloader.py::test_search_existing_files`
  - Note: `test_search_existing_files` fails only because the sandbox lacks `/dev/shm` (multiprocessing SemLock) — environmental, not a code bug. Deselect it as above.
- `uv run ruff check .` (CI-equivalent) succeeds.

## Risks / open questions
- 3.14 may surface a dep with no 3.14 wheel during `uv lock --upgrade`; if so, pin the last compatible version (do not drop functionality). praw 7.8.2 is pure-Python and expected to install on 3.14.
- ruff's broader rule set (UP/B/C4/SIM) may flag existing code; auto-fix safe ones, manually review the rest. This is expected cleanup, not a blocker.
- `formatting_check.yml` (tox + `mdl`) and `protect_master.yml` are intentionally kept per user choice; they remain usable after the tox.ini update.

## Out of scope
- praw 8 migration (needs Reddit creds / live validation) — unchanged.
- Live third-party downloader updates (imgur/redgifs/gfycat/yt-dlp) — unchanged.
