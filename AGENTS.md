# Development guide

This is automated-forecasting-monorepo: the engine and the scheduler under
`packages/`, the dashboards (AIRO, EEAI, RSI) under `dashboards/`, and submissions
(ForecastBench and LEAP) under `submissions/`. See `README.md` for the layout.
For a new dashboard, follow [registration, publication and scheduling](dashboards/AGENTS.md).

## Local setup and checks

Use Python 3.11+ (CI tests 3.11–3.13) and Node 22.x at 22.19 or newer with npm.
The root is a workspace, not an installable distribution. From the repo root:

```sh
python3 -m venv .venv
. .venv/bin/activate
python -m pip install -e './packages/forecaster[dev]' -e ./packages/dashboard \
    -e './dashboards/airo[test]' -e ./dashboards/eeai -e ./dashboards/rsi \
    -e ./submissions/forecastbench -e ./submissions/leap
npm ci --prefix packages/forecaster
```

Keep `.venv/` out of git (add `/.venv/` to `.git/info/exclude` if needed).
Use the checked-in npm lockfile: it pins Pi for the engine's integration tests.
Keep dependencies flowing from projects to the shared engine and dashboard kit;
don't import one dashboard or submission's code from another. Add tests in
the owning project's suite (the root `pyproject.toml` lists the test paths).

With the virtual environment active, run from the root:

```sh
mkdir -p .venv/tmp
TMPDIR="$PWD/.venv/tmp" PI_CODING_AGENT_DIR="$(mktemp -d "$PWD/.venv/tmp/pi.XXXXXX")" \
    python -m pytest -q -rs
node --test dashboards/airo/tests/test_tooltips.cjs
```

The test suite needs no provider credentials. Pi tests use a local scripted
model; the temporary agent directory isolates any existing coding login.
Check the skip summary: missing external research inputs can be expected;
missing Node or Pi means the engine tests didn't fully run.

## Build and preview without credentials

From `dashboards/airo`, run `python3 -m redlines build` then
`python3 -m redlines assemble`. These stdlib-only commands rebuild the result
blobs and `index.html` using committed inputs and the tracked Graph 3/4 mirrors
when external inputs are absent. On a clean checkout, `git status --short`
should stay empty, as checked by CI. Edit page sources under `web/demo/`, not
the assembled HTML directly.

For a local AIRO preview, serve just the assembled page and public image assets.
From `dashboards/airo`:

```sh
preview="$(mktemp -d)"
mkdir -p "$preview/web/img"
cp index.html "$preview/"
cp web/img/fri-logo.svg web/img/forecastbench-parity.png "$preview/web/img/"
python3 -m http.server 8000 --bind 127.0.0.1 --directory "$preview"
```

Open `http://127.0.0.1:8000/`. This preview does not stage the production
download or licensed fonts. For EEAI and RSI, use the synthetic previews in
their respective READMEs. Don't run acquisition, paid forecasts, scheduling,
or publishing jobs to test a local build.

## Production

Develop and test in the checkout. Production runs from a separate release.
When the user asks to deploy, merge through the repository's normal PR/check
requirements, then run `airo deploy`. Inspect `airo status` and verify the public
URL from the private runtime configuration before reporting success. A merge by
itself does not deploy. Normal deployment publishes the new frontend with the
existing published forecast snapshot and downloads; it does not require fresh
forecasts. Verify that release metadata names the new frontend commit and retains
the data snapshot hash. `airo deploy --publish-data` explicitly rebuilds forecast
data and requires the instrument integrity checks; scheduled forecast jobs also
validate before publishing. Never silently fall back after a data check fails.
`airo deploy --keep-publication` retains the entire old site and is only for an
explicit code-only deployment. Use `airo rollback` for an unhealthy release.

See `packages/ops/README.md` for schedules, run receipts and recovery. Add or change
scheduled work in `packages/ops/jobs.json`, then merge and deploy. Do not launch paid
forecast jobs merely to test deployment; `airo smoke` exercises the runner without
provider calls. ForecastBench rounds upload validated sets automatically; upload
failures page and can be retried without rerunning forecasts. See
`submissions/forecastbench/README.md` for the private authentication setup.

Keep machine names, personal paths, production data, credentials and deployment
receipts outside git. Read the cell's private runtime configuration for its
installation details. Never copy the coding-provider login into a forecasting
job's agent directory.

## Privacy

Never commit personally identifiable information (PII) to this repository or
include it in pull requests, issues, dashboard pages, or public downloads.
This includes personal contact details, participant identifiers that can be
linked back to a person, and identifying details in free-text responses.
Use fictional data in examples and test fixtures. Keep raw survey extracts
in private storage. Before sharing human forecast aggregates, check whether
small groups or combinations of attributes could identify a respondent.
