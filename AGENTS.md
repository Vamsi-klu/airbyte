# AGENTS.md

## Cursor Cloud specific instructions

This is the **Airbyte connectors monorepo**. There is **no single application** to run: each
connector under `airbyte-integrations/connectors/<connector-name>/` is an independent project,
and development happens per-connector. There is a stray root `poetry.lock` but **no root
`pyproject.toml`** — do not try to `poetry install` at the repo root.

### Toolchain (installed by the update script)

`uv`, `poetry`, `poe` (poethepoet) and the `airbyte-cdk` CLI are installed via `uv` into
`~/.local/bin`, which is on `PATH` for login shells via `~/.bashrc` (the uv installer appends
`. "$HOME/.local/bin/env"`). Python **3.11** is provided by uv (`uv python find 3.11`); note the
system Python is 3.12. If a fresh non-login shell can't find these tools, run
`export PATH="$HOME/.local/bin:$PATH"`.

### Per-connector workflow (Python connectors)

Task definitions live in `poe-tasks/*.toml`; run tasks from **inside a connector directory**
(or use `poe connector <name> <task>` / `poe source <shortname> <task>` from the repo root).

- Install deps: `poetry install --all-extras` (the connector's `poe install` also runs
  `uv tool install --upgrade 'airbyte-cdk[dev]'`).
- Set `POETRY_VIRTUALENVS_IN_PROJECT=true` for a deterministic in-project `.venv` (matches
  `.devcontainer`).
- Some connectors pin Python `^3.11`; if `poetry install` picks the wrong interpreter, run
  `poetry env use "$(uv python find 3.11)"` first.
- Lint / format: `poe check-ruff-lint`, `poe check-ruff-format` (ruff config in `ruff.toml`).
  `poe check-mypy` runs but some connectors have **pre-existing** mypy errors in their own code;
  ruff is what pre-commit enforces (`.pre-commit-config.yaml`).
- Unit tests: `poe test-unit-tests` (skips gracefully if no `unit_tests/` dir).
- Run the connector as an app: `poetry run <connector-name> spec|check|discover|read ...`
  (e.g. `poetry run source-hardcoded-records read --config integration_tests/sample_config.json
  --catalog integration_tests/configured_catalog.json`).

### Secrets for tests

Integration / FAST "Airbyte standard" tests may look for `secrets/config.json` (the `secrets/`
dir is gitignored). Normally this is populated by `airbyte-cdk secrets fetch`, which requires
Airbyte's Google Secret Manager credentials that are not available here. `source-hardcoded-records`
is a good credential-free connector to validate the environment — its config is just
`{"count": N}`, so you can create `secrets/config.json` locally to make its standard tests pass.

### Other connector types

- **Java/Kotlin** connectors use Gradle via the root wrapper `./gradlew` (Gradle 8.14, Java 21 is
  installed). Use `poe gradle <task>` from the connector dir. First run warms the Gradle cache and
  is slow.
- **Manifest-only** connectors keep unit tests in `unit_tests/` with their own `pyproject.toml`;
  `poe test-unit-tests` installs and runs them.

### Docker

Docker is **not** installed and is **not** needed for local dev (install/lint/test/run). It is
only required to build connector images (`airbyte-cdk image build`) or run container-based
acceptance tests.
