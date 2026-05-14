# devassure-action

> [!IMPORTANT] DevAssure O2 automatically generate & run e2e tests on browsers for every GitHub pull request

**DevAssure O2 reads your PR, generates the right tests, and runs them before merge. No scripts. No maintenance.**

Works on every commit, branch or pull request.

Action uses the DevAssure CLI - [@devassure/cli](https://www.npmjs.com/package/@devassure/cli).

## How it works

> [!IMPORTANT] reads code diff → maps blast radius → generates tests → executes

1. Add the devassure-action to your GitHub Actions workflow and congiure to run it on every pull request.
2. The action invokes the DevAssure O2 agent which reads the code diff and understands the changes.
3. Generates extensive end to end UI tests (natural language) to validate the code changes.
4. Executes the natural language tests on browsers.
5. Finds the bugs and reports them to the PR along with reports.

## Before vs After DevAssure O2

**Before:**  
❌ Write & maintain test scripts  
❌ Run full regression suite  
❌ Miss edge cases  

**After DevAssure O2:**  
✅ Tests generated from PR diff  
✅ Only impacted areas tested  
✅ Bugs caught before merge  

Learn more about DevAssure O2 [here](https://devassure.io).

## What the action does

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
  - adds test result details to the GitHub Actions run summary (using [dorny/test-reporter](https://github.com/dorny/test-reporter)).

## Create a DevAssure token

1. Log in to [https://app.devassure.io](https://app.devassure.io) OR Sign up for free at [https://app.devassure.io/sign_up](https://app.devassure.io/sign_up).
2. Open your account/settings token section.
3. Generate a new API token.
4. Save it as a GitHub repository secret named `DEVASSURE_TOKEN`.

## Prerequisites

- This action supports Unix-based runners only (for example, Linux and macOS).
- In the `actions/checkout` step, `with.ref` is required so the agent can resolve and use the code diff.

## Usage

### Default (test pull request / branch)

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
    minimum_score: 80
    workers: 2
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
    workers: 2
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
| `command` | `test` | DevAssure command to execute: `setup`, `test`, `run`, `summary`, `archive`, or `archive-report` |
| `token` | _empty_ | DevAssure API token. If unset, falls back to `DEVASSURE_TOKEN` from environment |
| `path` | _empty_ | Relative project path to run from (useful when tests are in a subdirectory) |
| `source` | _empty_ | Source branch used by `test` command scope and branch checkout logic |
| `target` | _empty_ | Target branch used by `test` command scope and comparison baseline |
| `commit_id` | _empty_ | Commit SHA used by `test` command scope |
| `filter` | _empty_ | Filter expression for narrowing `run` command execution |
| `query` | _empty_ | Query string for selecting tests in `run` command |
| `tag` | _empty_ | Tag-based selector for `run` command |
| `priority` | _empty_ | Priority-based selector for `run` command |
| `folder` | _empty_ | Folder selector for `run` command |
| `url` | _empty_ | Application URL under test for `test` and `run` commands |
| `headless` | `true` | Headless browser mode flag for `test` and `run` |
| `session_id` | _empty_ | Specific session id for `summary` or `archive`/`archive-report` (defaults to latest session when empty) |
| `archive` | `true` | For `test`/`run`, set to `false` to skip `archive-report --last` and artifact archiving |
| `minimum_score` | `75` | Minimum score threshold for `test`/`run`; job fails when summary score is below this value |
| `workers` | _empty_ | Worker count for parallel execution in `test`/`run` (must be integer greater than `0`) |
| `environment` | _empty_ | Environment name passed to `test`/`run` (for example `staging`, `qa`, or `production`) |
| `commit_tests` | `false` | When `true`, commits generated tests back to the branch (only applies to the `test` command). Requires `contents: write` permission |

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

## Committing generated tests

When the `test` command generates new or updated tests, you can have the action automatically commit them back to your branch by setting `commit_tests: true`.

Commits are authored by `devassure[bot] <devassure[bot]@users.noreply.github.com>` with the message:

```
chore: add DevAssure generated tests [skip actions]
```

### Requirements

- Only works with the `test` command.
- Your workflow must grant `contents: write` permission. Without it the push step will fail with an error.
- Fork pull requests are skipped to prevent unintended writes to external repositories.

### Example workflow

```yaml
name: devassure
on: [pull_request]

permissions:
  contents: write

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
        with:
          commit_tests: true
        env:
          DEVASSURE_TOKEN: ${{ secrets.DEVASSURE_TOKEN }}
```

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

## FAQs

### How does minimum score work?

The minimum score is the minimum value that the tests need to achieve to pass. Default is `75`. If the score is less than the minimum score, the job will fail. If the score is greater than the minimum score, the job will pass. Set `minimum_score` to `0` to disable score validation.

### How is credits consumed?

Credits are consumed based on the number of browser interactions done and the complexity of the tests. The usage for each execution can be see in the DevAssure web portal - [/usage](https://app.devassure.io/usage).

### How can I view the complete report?

The complete report is archived and can be downlaoded from the GitHub Actions run summary. The zip file can be opened using DevAssure [CLI](https://www.npmjs.com/package/@devassure/cli) or DevAssure [VSCode extension](https://marketplace.visualstudio.com/items?itemName=devassure.devassure-vscode).

### Can I invoke the agent in my local machine?

Yes, you can invoke the agent in your local machine using the DevAssure CLI npm package - [@devassure/cli](https://www.npmjs.com/package/@devassure/cli).