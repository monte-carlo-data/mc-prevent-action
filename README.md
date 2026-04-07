# MC Prevent — GitHub Action

Assess data pipeline risk before merge. This action calls the Monte Carlo MC Prevent API to evaluate pull request changes against your data observability signals (alerts, lineage, monitor coverage) and returns a pass/warn/fail verdict.

## Quick Start

### 1. Create Monte Carlo API keys

Go to **Monte Carlo → Settings → API Keys → Create Key**. Save the key ID and token.

### 2. Add repository secrets

Go to **GitHub repo → Settings → Secrets and variables → Actions → New repository secret**. Add:

| Secret | Value |
|---|---|
| `MCD_ID` | Your Monte Carlo API key ID |
| `MCD_TOKEN` | Your Monte Carlo API key token |

### 3. Add the workflow

Create `.github/workflows/mc-prevent.yml`:

```yaml
name: MC Prevent
on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    branches: [main]

jobs:
  mc-prevent:
    if: github.event.pull_request.draft == false
    runs-on: ubuntu-latest
    steps:
      - uses: monte-carlo-data/mc-prevent-action@v1
        with:
          mcd-id: ${{ secrets.MCD_ID }}
          mcd-token: ${{ secrets.MCD_TOKEN }}
```

That's it. MC Prevent runs on every pull request (skips drafts) and reports a verdict.

## Parameters

| Input | Required | Default | Description |
|---|---|---|---|
| `mcd-id` | Yes | — | Monte Carlo API Key ID |
| `mcd-token` | Yes | — | Monte Carlo API Key Token |
| `api-url` | No | `https://api.getmontecarlo.com/ci/assess` | API endpoint URL |
| `fail-on-error` | No | `true` | Whether warn/fail verdicts cause the step to exit non-zero |
| `poll-interval` | No | `30` | Seconds between poll attempts |
| `max-wait` | No | `300` | Maximum seconds to wait for assessment |

### Example with custom parameters

```yaml
- uses: monte-carlo-data/mc-prevent-action@v1
  with:
    mcd-id: ${{ secrets.MCD_ID }}
    mcd-token: ${{ secrets.MCD_TOKEN }}
    fail-on-error: "false"
    poll-interval: "15"
    max-wait: "120"
```

## Outputs

| Output | Description |
|---|---|
| `conclusion` | `pass`, `warn`, or `fail` |
| `risk-tier` | `high`, `medium`, `low`, or `none` |
| `assessment-source` | `pr_agent`, `override`, `none`, or `error` |
| `response` | Full JSON response |

### Using outputs

```yaml
- uses: monte-carlo-data/mc-prevent-action@v1
  id: gate
  with:
    mcd-id: ${{ secrets.MCD_ID }}
    mcd-token: ${{ secrets.MCD_TOKEN }}

- run: echo "Risk tier is ${{ steps.gate.outputs.risk-tier }}"
```

## How it works

1. Triggered on pull request events (skips draft PRs, runs on ready for review)
2. Calls Monte Carlo's `/ci/assess` API with the repo, PR number, and commit SHA
3. If no assessment is available yet (the PR agent may still be analyzing), waits up to `max-wait` seconds
4. If a cached verdict from a previous commit exists, reuses it immediately
5. Displays the verdict and a human-readable summary in the job output and step summary
6. Raw API response available in a collapsible section

### Verdicts

MC Prevent returns one of three verdicts based on the risk assessment:

| Verdict | What it means | CI job behavior (`fail-on-error: true`) | Check run on PR |
|---|---|---|---|
| **pass** | No significant risk detected | Job passes (green) | Green |
| **warn** | Risk detected — review recommended | Job fails (red) | Grey (neutral) |
| **fail** | High risk — merge not recommended | Job fails (red) | Red |

**Note on CI job vs check run:** The CI job can only show green or red. The "MC Prevent CI Gate Result" check run posted on the PR shows the actual severity — green for pass, grey for warn, red for fail. If you configure branch protection, require the check run (not the CI job) for accurate gating.

### What `fail-on-error` controls

| Setting | Behavior |
|---|---|
| `fail-on-error: true` (default) | Warn and fail both cause the step to exit non-zero (red). This draws attention to risks. |
| `fail-on-error: false` | The step always passes (green). The verdict is only visible in the job output, step summary, and the check run on the PR. Use this for a silent, non-blocking setup. |

### Missing credentials

If `MCD_ID` or `MCD_TOKEN` are not set, MC Prevent skips silently (step passes). Adding the action before configuring credentials won't break your CI.

## Override

Add the `mc-override` label to your pull request to bypass MC Prevent.

- The verdict returns **pass** regardless of risk
- The "MC Prevent CI Gate Result" check run on the PR immediately flips to green — no commit or CI re-run needed
- All overrides are logged for audit

## Troubleshooting

**MC Prevent times out with no assessment:**
The PR agent posts its assessment when a PR is opened or marked ready for review. If you push additional commits, MC Prevent reuses the cached verdict from the initial assessment. If no assessment exists after `max-wait` seconds, the step exits without blocking.

**Authentication errors (401):**
Verify that `MCD_ID` and `MCD_TOKEN` are set correctly as repository secrets.

**CI job shows red but the change is low risk:**
Check the step output — if the verdict is `warn` (not `fail`), this is expected when `fail-on-error: true`. The check run on the PR will show grey (neutral), not red. Set `fail-on-error: false` if you don't want warnings to fail the CI job.

## Resources

- [Monte Carlo Documentation](https://docs.getmontecarlo.com)
- [MC Prevent Overview](https://docs.getmontecarlo.com/docs/github)
- [Source Code](https://github.com/monte-carlo-data/mc-prevent-action)
