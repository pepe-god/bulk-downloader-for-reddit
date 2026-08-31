# AGENTS.md

Compact guidance for agents working on the Bulk Downloader for Reddit (`bdfr`).

## Toolchain
- Dependency management is **uv**, not pip. A committed `uv.lock` pins exact versions.
  - `uv sync --all-extras` — create `.venv` and install everything (including dev extras).
  - `uv run <cmd>` — run anything inside the env (e.g. `uv run pytest`, `uv run ruff check bdfr`).
  - Add/upgrade a dep in `pyproject.toml`, then `uv lock --upgrade`.
- Minimum Python is **3.14** (`requires-python = ">=3.14"`); the test suite targets 3.14. Do not reintroduce 3.9/3.10/3.11/3.12/3.13 support.
- Lint/formatter config lives **in `pyproject.toml`** (`[tool.ruff]` for lint + import sorting, `[tool.black]` for formatting). There is no `.flake8`. ruff/black line-length is 120 (not 79); ruff `target-version = "py314"`, black `target-version = ["py314"]`.

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
- Authenticated/reddit tests need a Reddit token: write a `test_config.cfg` next to `bdfr/default_config.cfg` with `user_token = <token>`.
- Lint with `uv run ruff check bdfr` for the full pyproject-based config.

## References
- Architecture: `docs/ARCHITECTURE.md`. Contribution/style guide and dev setup: `docs/CONTRIBUTING.md`. Tox: `tox.ini`.
