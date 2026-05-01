# MC Prevent — GitHub Action

Assess data pipeline risk before merge. This action calls the Monte Carlo MC Prevent API to evaluate pull request changes against your data observability signals (alerts, lineage, monitor coverage) and returns a pass/fail verdict.

## Quick Start

### 1. Create Monte Carlo API keys

Go to **Monte Carlo → Settings → API Keys → Create Key**. Save the key ID and token.

### 2. Add repository secrets

Go to **GitHub repo → Settings → Secrets and variables → Actions → New repository secret**. Add:

| Secret      | Value                          |
| ----------- | ------------------------------ |
| `MCD_ID`    | Your Monte Carlo API key ID    |
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

| Input           | Required | Default                                   | Description                                                                                                              |
| --------------- | -------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `mcd-id`        | Yes      | —                                         | Monte Carlo API Key ID                                                                                                   |
| `mcd-token`     | Yes      | —                                         | Monte Carlo API Key Token                                                                                                |
| `api-url`       | No       | `https://api.getmontecarlo.com/ci/assess` | API endpoint URL                                                                                                         |
| `block-on`      | No       | _(UI setting)_                            | Which risk tiers exit non-zero: `low+`, `medium+`, `high+`, or `none`. When unset, the Monte Carlo UI's setting applies. |
| `poll-interval` | No       | `30`                                      | Seconds between poll attempts                                                                                            |
| `max-wait`      | No       | `300`                                     | Maximum seconds to wait for assessment                                                                                   |

> **Migrating from `fail-on`:** `fail-on` is deprecated but still works for backward compatibility. `block-on` takes precedence when both are set. Mapping: `warn_and_fail` → `medium+`, `fail_only` → `high+`, `none` → `none`.

> **Migrating from `fail-on-error`:** `fail-on-error: false` is equivalent to `block-on: none`. `fail-on-error: true` is a no-op (defers to `block-on`). The `fail-on-error` parameter still works for backward compatibility.

### Example: only block on high-risk PRs

```yaml
- uses: monte-carlo-data/mc-prevent-action@v1
  with:
    mcd-id: ${{ secrets.MCD_ID }}
    mcd-token: ${{ secrets.MCD_TOKEN }}
    block-on: high+
```

## Outputs

| Output              | Description                                |
| ------------------- | ------------------------------------------ |
| `conclusion`        | `pass` or `fail`                           |
| `risk-tier`         | `high`, `medium`, `low`, or `none`         |
| `assessment-source` | `pr_agent`, `override`, `none`, or `error` |
| `response`          | Full JSON response                         |

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
5. Displays the verdict and a summary in the CI job output and step summary
6. The same verdict appears on the "MC Prevent CI Gate Result" check run on the PR
7. Raw API response available in a collapsible section

### Verdicts and `block-on`

MC Prevent assigns a risk tier (low, medium, high) to each PR based on a rubric that evaluates schema changes, data volume impact, downstream exposure, and other factors. The `block-on` parameter determines which tiers cause the CI step to fail:

| Risk tier  | `low+` | `medium+` | `high+` | `none` |
| ---------- | ------ | --------- | ------- | ------ |
| **low**    | Fail   | Pass      | Pass    | Pass   |
| **medium** | Fail   | Fail      | Pass    | Pass   |
| **high**   | Fail   | Fail      | Fail    | Pass   |

**Note on `block-on` default:** When `block-on` is not set in the workflow, the backend uses the setting configured in the Monte Carlo UI. This means teams can change the policy from the UI without modifying CI configuration.

### Behavior by setup stage

MC Prevent is designed for progressive adoption — it never blocks your CI due to incomplete setup. You can add the workflow file first and configure the remaining pieces at your own pace.

| Setup stage                                  | CI result    | What you'll see                                                         |
| -------------------------------------------- | ------------ | ----------------------------------------------------------------------- |
| Workflow added, secrets not yet configured   | Pass (green) | Step skips instantly — no API call is made                              |
| Secrets configured, PR agent not yet enabled | Pass (green) | Step polls for up to `max-wait` seconds, then passes with no assessment |
| Secrets configured, PR agent enabled         | Pass / Fail  | Full risk verdict based on the PR agent's analysis                      |

**Tip:** To avoid the polling wait in stage two, enable the PR agent in **Monte Carlo → Settings → AI Agents** before (or shortly after) adding your API credentials.

## Controlling which branches are gated

Use the workflow trigger to control which PRs run MC Prevent. By default, the quick start template runs on PRs targeting `main`:

```yaml
on:
  pull_request:
    branches: [main]
```

To gate additional branches:

```yaml
on:
  pull_request:
    branches: [main, release/*, production]
```

To run on all PRs (not recommended for high-traffic repos):

```yaml
on:
  pull_request:
```

## Override

Add the `mc-override` label to your pull request to bypass MC Prevent.

- The verdict returns **pass** regardless of risk
- The "MC Prevent CI Gate Result" check run on the PR immediately flips to green — no commit or CI re-run needed
- All overrides are logged for audit

## Troubleshooting

**MC Prevent times out with no assessment:**
The PR agent posts its assessment when a PR is opened or marked ready for review. If you push additional commits, MC Prevent reuses the cached verdict from the initial assessment. If no assessment exists after `max-wait` seconds, the step passes without blocking. This typically means the PR agent is not yet enabled — see the [setup stages](#behavior-by-setup-stage) table above.

**Authentication errors (401):**
Verify that `MCD_ID` and `MCD_TOKEN` are set correctly as repository secrets.

**CI job shows red but the change is low risk:**
Check the step output — the risk tier might be medium, which fails when `block-on` is set to `medium+` (either explicitly or via the Monte Carlo UI). Set `block-on: high+` to only block on high-risk PRs, or `block-on: none` if you don't want any verdict to fail the CI job. You can also change this in the Monte Carlo UI under Settings → Integrations → CI Gate.

## Resources

- [Monte Carlo Documentation](https://docs.getmontecarlo.com)
- [MC Prevent Overview](https://docs.getmontecarlo.com/docs/github)
- [Source Code](https://github.com/monte-carlo-data/mc-prevent-action)
