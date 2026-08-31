# BDFR — Safe Modernization Plan

## Context
- Last upstream commit: **2023-01-31** (~3.5 years stale). The codebase predates current
  library/stdlib norms for a Python 3.11 project.
- Offline test baseline (verified this session): **179 passed, 4 skipped**, exactly one
  `DeprecationWarning` (`importlib.resources.path` in `bdfr/connector.py:205`).
- Scope agreed with user: **safe, offline-verifiable modernization only**. Riskier items
  (praw 8, live third-party downloaders) are listed separately and flagged for manual/
  online validation — included in the plan but explicitly out of primary implementation scope.
- Dependency versions are already current (prior `uv` migration ran `uv lock --upgrade`;
  praw pinned `<8` on purpose). No version bumps needed beyond what this plan changes.

## Constraints
- Minimum Python stays **3.11** — this is what makes PEP 604 unions and
  `importlib.resources.files()` valid, so keep `requires-python = ">=3.11"`.
- **Do NOT** bump `praw` to `>=8`. `bdfr/oauth2.py` uses
  `praw.util.token_manager.BaseTokenManager` and `prawcore.auth.BaseAuthorizer`, both removed
  in praw 8. Migrating is a separate, high-risk task (see Flagged section).

## Tasks (ordered)

### 1. Replace `appdirs` with `platformdirs`
`appdirs` is unmaintained (last release 2020); `platformdirs` is the maintained drop-in.
- `pyproject.toml`: remove `"appdirs>=1.4.4"`, add `"platformdirs>=4.0.0"`.
- `bdfr/connector.py`:
  - line 56 `self.config_directories = appdirs.AppDirs("bdfr", "BDFR")` →
    `self.config_directories = platformdirs.user_config_dir("bdfr", "BDFR")`
    (this returns the directory string directly).
  - line 180 `self.config_directory = Path(self.config_directories.user_config_dir)` →
    `self.config_directory = Path(self.config_directories)` (attribute no longer exists).
  - replace `import appdirs` with `import platformdirs`.
- `bdfr/completion.py`:
  - line 15 `appdirs.user_data_dir()` → `platformdirs.user_data_dir()`.
  - replace `import appdirs` with `import platformdirs`.
- Run `uv lock --upgrade` so `uv.lock` records `platformdirs`.

### 2. Fix deprecated `importlib.resources.path`
`bdfr/connector.py:205` triggers a `DeprecationWarning`.
- Replace:
  ```python
  with importlib.resources.path("bdfr", "default_config.cfg") as path:
  ```
  with:
  ```python
  with importlib.resources.as_file(
      importlib.resources.files("bdfr").joinpath("default_config.cfg")
  ) as path:
  ```
  `as_file` yields a real `pathlib.Path`, so the existing `shutil.copy(...)` and
  `self.cfg_parser.read(self.config_location)` calls need no further change.

### 3. Typing modernization to PEP 604
Code already uses builtin `list[...]` but still imports `Optional`/`Union` from `typing`
(20+ files). Standardize on `X | None` / `A | B`.
- One-off automated pass (no need to add to deps; run via uvx):
  `uvx pyupgrade --py311-plus bdfr`
  This converts `Optional[X]`→`X | None`, `Union[A,B]`→`A | B`, and drops now-unused
  `from typing import ...` lines.
- Then format: `uv run black .` and `uv run isort .`.
- Spot-check that only genuinely-typing-only imports were removed; keep `typing` imports that
  are still used (e.g. `Callable`, `Iterable`, `Any`, `TYPE_CHECKING`).

### 4. (Optional, minor) Drop obsolete `# -*- coding: utf-8 -*-` headers
Present in a few files (e.g. `completion.py`); unnecessary on Python 3. Skip if it adds churn.

### 5. Re-sync and re-verify lockfile
- `uv sync --all-extras` (already done this session; re-run after deps change in step 1).

## Validation (all offline)
- `uv run flake8 bdfr` (in a constrained container without `/dev/shm`, add `--jobs 1`).
- `uv run black --check . && uv run isort --check-only .`
- `uv run pytest -m "not online and not reddit and not authenticated" \
   --deselect tests/test_downloader.py::test_search_existing_files`
  Expected: **179 passed, 0 skipped-from-scope, and the `importlib.resources.path`
  DeprecationWarning gone.**
- Sanity: `uv run bdfr --version` still works (dynamic version from `bdfr/__init__.py`).

## Risks
- `as_file` requires an extractable resource; `default_config.cfg` is a packaged data-file, so
  it resolves to a real path — safe.
- `platformdirs.user_data_dir()` with no args returns the OS default user-data dir, matching
  prior `appdirs.user_data_dir()` behavior.
- `pyupgrade` touches many files; the full offline suite must be re-run (not just the changed
  module) to catch collateral.

## Flagged for manual / online validation (NOT primary scope)
These are real staleness risks for a 2023-era codebase but **cannot be verified offline** in
this environment. Keep them in the plan as follow-up items requiring network + Reddit creds:
- **praw 8 migration**: rewrite `bdfr/oauth2.py` token persistence (replace
  `BaseTokenManager`/`prawcore.auth.BaseAuthorizer`), then drop the `praw<8` cap. Validate with
  `uv run pytest -m "authenticated or reddit"`.
- **Live third-party downloaders**: `imgur.py` (hardcoded Imgur Client-ID `546c25a59c58ad7`),
  `redgifs.py` (v2 API may have changed), `gfycat.py` (redirects to redgifs), and
  `youtube.py`/`ytdlp_fallback.py` (yt-dlp). Exercise via the `reddit`/`online` test markers
  after `devscripts/configure.sh` + `REDDIT_TOKEN`.
- **`user_agent = socket.gethostname()`** in `connector.py` may be rejected by Reddit's current
  UA policy; consider a descriptive static UA (behavioral change → validate online first).
