# AGENTS.md

Compact guidance for agents working on the Bulk Downloader for Reddit (`bdfr`).

## Toolchain
- Dependency management is **uv**, not pip. A committed `uv.lock` pins exact versions.
  - `uv sync --all-extras` — create `.venv` and install everything (including dev extras).
  - `uv run <cmd>` — run anything inside the env (e.g. `uv run pytest`, `uv run flake8 bdfr`).
  - Add/upgrade a dep in `pyproject.toml`, then `uv lock --upgrade`.
- Minimum Python is **3.11** (`requires-python = ">=3.11"`); CI tests 3.11/3.12/3.13. Do not reintroduce 3.9/3.10 support.
- Lint/formatter config lives **in `pyproject.toml`** (`[tool.flake8]` via Flake8-pyproject, `[tool.black]`, `[tool.isort]`). There is no `.flake8`. flake8 line-length is 120 (not 79); black/isort also target 120 with `profile = "black"`, `py_version = 311`.

## Dependencies — hard constraints
- **Do NOT bump `praw` to `>=8`.** It is pinned `praw>=7.2.0,<8` on purpose: `bdfr/oauth2.py` imports `praw.util.token_manager.BaseTokenManager` and uses `prawcore.auth.BaseAuthorizer`, both removed in praw 8. Migrating to praw 8 requires rewriting the token-persistence logic in `oauth2.py` first.
- `prawcore>=2.0.0` is an explicit dependency (used directly in `oauth2.py`), not just transitive.

## Build / packaging
- Version is **dynamic**: setuptools reads `bdfr.__version__` (defined in `bdfr/__init__.py`). Do not remove that attribute — the build needs it.
- Four console scripts, all in `bdfr/__main__.py`: `bdfr` → `cli`, `bdfr-download` → `cli_download`, `bdfr-archive` → `cli_archive`, `bdfr-clone` → `cli_clone`. The `download`/`archive`/`clone` distinction is the core CLI surface.
- `bdfr/default_config.cfg` is installed as a data-file; `OAuth2TokenManager` persists the refresh token into BDFR's own configparser config.

## Testing
- pytest markers (declared, `--strict-markers`): `online`, `reddit`, `slow`, `authenticated`.
- Run the offline/local suite with: `uv run pytest -m "not online and not reddit and not authenticated"`.
- Authenticated/reddit tests need a Reddit token: `devscripts/configure.sh` (or `configure.ps1` on Windows) writes config from the `REDDIT_TOKEN` secret (see `.github/workflows/test.yml`).
- The test config in CI lints only syntax-critical flake8 codes (`--select=E9,F63,F7,F82`); locally run `uv run flake8 bdfr` for the full pyproject-based config.

## References
- Architecture: `docs/ARCHITECTURE.md`. Contribution/style guide and dev setup: `docs/CONTRIBUTING.md`. Pre-commit: `.pre-commit-config.yaml`. Tox: `tox.ini`.
