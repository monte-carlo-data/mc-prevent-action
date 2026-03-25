# Monte Carlo Safety Gate

A GitHub Action that runs Monte Carlo's CI Safety Gate on pull requests to assess the risk of data pipeline changes before merge.

## Quick Start

Add to your workflow (`.github/workflows/mc-safety-gate.yml`):

```yaml
name: MC Safety Gate
on:
  pull_request:
    branches: [main]

jobs:
  safety-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: monte-carlo-data/mcd-ci-gate-agent@v1
        with:
          mc-api-key: ${{ secrets.MC_API_KEY }}
```

## How It Works

1. Triggered on pull request events
2. Calls Monte Carlo's `POST /ci/assess` API with PR metadata
3. MC retrieves the risk assessment and applies gate policy
4. Posts an "MC Safety Gate" Check Run on the PR
5. Returns pass/warn/fail — `fail` blocks merge (if branch protection requires it)

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `mc-api-key` | Yes | — | Monte Carlo API key |
| `api-url` | No | `https://api.getmontecarlo.com/ci/assess` | API endpoint URL |
| `fail-on-error` | No | `true` | Set to `false` to never fail the step |

## Outputs

| Output | Description |
|---|---|
| `conclusion` | `pass`, `warn`, or `fail` |
| `risk-tier` | `high`, `medium`, `low`, or `none` |
| `assessment-source` | `pr_agent`, `override`, `none`, or `error` |
| `response` | Full JSON response |

### Using outputs

```yaml
- uses: monte-carlo-data/mcd-ci-gate-agent@v1
  id: gate
  with:
    mc-api-key: ${{ secrets.MC_API_KEY }}

- run: echo "Risk tier is ${{ steps.gate.outputs.risk-tier }}"
```

## Override

Add the `mc-override` label to a PR to bypass the gate. The gate returns `pass` with a note that the override is active. Overrides are logged for audit.

## Gate Modes

Configured per account in Monte Carlo:

| Mode | Behavior |
|---|---|
| `warn_only` (default) | Never blocks. High-risk changes show as warnings. |
| `fail_on_high_risk` | High-risk changes fail the check and block merge. |

## Setup

1. Get your Monte Carlo API key
2. Add it as a repository secret named `MC_API_KEY`
3. Add the workflow file above
4. Open a PR — the safety gate runs automatically
