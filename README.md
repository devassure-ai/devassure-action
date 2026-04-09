# devassure-action

GitHub Action to run [@devassure/cli](https://www.npmjs.com/package/@devassure/cli) commands in CI.

## What this action does

- Installs `@devassure/cli` globally.
- Prints `devassure version`.
- Configures token from:
  - `with.token`, or
  - `env.DEVASSURE_TOKEN` (for example `${{ secrets.DEVASSURE_TOKEN }}`).
- Runs `devassure <command>` with command-specific optional arguments.
- For `test` and `run`:
  - runs `devassure summary --last`
  - validates `score` from `devassure summary --last` against `minimum_score` (default `75`)
  - optionally archives report and uploads artifact.

## Create a DevAssure token

1. Log in to [https://app.devassure.io](https://app.devassure.io).
2. Open your account/settings token section.
3. Generate a new API token.
4. Save it as a GitHub repository secret named `DEVASSURE_TOKEN`.

## Prerequisites

- This action supports Unix-based runners only (for example, Linux and macOS).
- In the `actions/checkout` step, `with.ref` is required so the agent can resolve and use the code diff.

## Usage

### Minimal (defaults to `test`)

```yaml
name: devassure
on: [push]

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          ref: ${{ github.head_ref || github.ref_name }}
      - uses: actions/setup-node@v4
        with:
          node-version: "24"
      - name: Run DevAssure
        uses: devassure-ai/devassure-action@v1
        env:
          DEVASSURE_TOKEN: ${{ secrets.DEVASSURE_TOKEN }}
```

### Test command with inputs

```yaml
- name: Run DevAssure test
  uses: devassure-ai/devassure-action@v1
  with:
    command: test
    minimum_score: "80"
    workers: "2"
    path: e2e/tests
    source: staging
    target: production
    commit_id: ${{ github.sha }}
    url: https://example.com
    environment: staging
    headless: "true"
  env:
    DEVASSURE_TOKEN: ${{ secrets.DEVASSURE_TOKEN }}
```

### Run command with filters

```yaml
- name: Run DevAssure run
  uses: devassure-ai/devassure-action@v1
  with:
    command: run
    workers: "2"
    path: e2e
    filter: smoke
    query: "login flow"
    tag: nightly
    priority: high
    folder: reports
    url: https://example.com
    environment: staging
    headless: "false"
  env:
    DEVASSURE_TOKEN: ${{ secrets.DEVASSURE_TOKEN }}
```

### Summary command (session-specific)

```yaml
- name: Run DevAssure summary
  uses: devassure-ai/devassure-action@v1
  with:
    command: summary
    session_id: sess_123
  env:
    DEVASSURE_TOKEN: ${{ secrets.DEVASSURE_TOKEN }}
```

If `session_id` is not set for `summary`, the action runs `devassure summary --last`.

### Archive command

```yaml
- name: Run DevAssure archive-report
  uses: devassure-ai/devassure-action@v1
  with:
    command: archive
    session_id: sess_123
  env:
    DEVASSURE_TOKEN: ${{ secrets.DEVASSURE_TOKEN }}
```

If `session_id` is not set for `archive` (or `archive-report`), the action runs `devassure archive-report --last`.

### Pass token directly

```yaml
- name: Run DevAssure with explicit token
  uses: devassure-ai/devassure-action@v1
  with:
    token: ${{ secrets.DEVASSURE_TOKEN }}
    command: setup
```

## Inputs

All inputs are optional.

| Input | Default | Description |
| --- | --- | --- |
| `command` | `test` | Command to run: `setup`, `test`, `run`, `summary`, `archive`, `archive-report` |
| `token` | _empty_ | DevAssure token. Fallback is `env.DEVASSURE_TOKEN` |
| `path` | _empty_ | Used by `test`, `run` |
| `source` | _empty_ | Used by `test` |
| `target` | _empty_ | Used by `test` |
| `commit_id` | _empty_ | Used by `test` |
| `filter` | _empty_ | Used by `run` |
| `query` | _empty_ | Used by `run` |
| `tag` | _empty_ | Used by `run` |
| `priority` | _empty_ | Used by `run` |
| `folder` | _empty_ | Used by `run` |
| `url` | _empty_ | Used by `test`, `run` |
| `headless` | `true` | Used by `test`, `run`; always passed to CLI |
| `session_id` | _empty_ | Used by `summary`, `archive`, `archive-report` |
| `archive` | `true` | For `test`/`run`, if not `false`, runs `archive-report --last` and uploads artifact |
| `minimum_score` | `75` | For `test`/`run`, final score gate from `devassure summary --last`; if `<= 0` or non-numeric, score check is skipped |
| `workers` | _empty_ | Number of parallel browser workers. if provided must be an integer greater than 0 |
| `environment` | _empty_ | Environment name for `test` and `run` (for example `staging`, `qa`, `production`) |

## Command parameter mapping

The action forwards supported parameters in this format:

`--<arg-name>="<arg-value>"`

- `setup`: none
- `test`: `path`, `source`, `target`, `commit_id`, `url`, `workers`, `environment`, `headless` (`headless` defaults to `true` and is always passed)
- `run`: `path`, `filter`, `query`, `tag`, `priority`, `folder`, `url`, `workers`, `environment`, `headless` (`headless` defaults to `true` and is always passed)
- `summary`: `session_id` when provided, else `--last` (never both)
- `archive` / `archive-report`: `session_id` when provided, else `--last` (never both)

For `test` and `run`, the action runs a final score check using `devassure summary --last`:
- If `minimum_score` is non-numeric or `<= 0`, score validation is skipped.
- If `score` is missing or `N/A`, the action fails.
- If `score` is lower than `minimum_score`, the action fails with:
  `Test score '<score>' is less than the minumum expected score (<minimum_score>)`

## Outputs

| Output | Description |
| --- | --- |
| `archive_path` | Archived report path parsed from `devassure archive-report` output |

## Notes

- Set this in your workflow to enable secret fallback:

```yaml
env:
  DEVASSURE_TOKEN: ${{ secrets.DEVASSURE_TOKEN }}
```

- For `test` and `run`, archive artifact upload is enabled by default (`archive: true`).

## Runner sizing recommendation

- Recommended baseline for parallel browser execution: `4 vCPU / 16 GB RAM`.
- If you increase worker count (or browser concurrency), increase machine size accordingly to avoid CPU and memory contention.
