# MC Prevent — GitHub Action

Assess data pipeline risk before merge. This action calls the Monte Carlo MC Prevent API to evaluate pull request changes against your data observability signals (alerts, lineage, monitor coverage) and returns a pass/warn/fail verdict.

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
| `fail-on`       | No       | _(UI setting)_                            | Which verdicts exit non-zero: `warn_and_fail`, `fail_only`, or `none`. When unset, the Monte Carlo UI's setting applies. |
| `poll-interval` | No       | `30`                                      | Seconds between poll attempts                                                                                            |
| `max-wait`      | No       | `300`                                     | Maximum seconds to wait for assessment                                                                                   |

> **Migrating from `fail-on-error`:** `fail-on-error: false` is equivalent to `fail-on: none`. `fail-on-error: true` is a no-op (defers to `fail-on`). The `fail-on-error` parameter still works for backward compatibility.

### Example: only block on fail, not warnings

```yaml
- uses: monte-carlo-data/mc-prevent-action@v1
  with:
    mcd-id: ${{ secrets.MCD_ID }}
    mcd-token: ${{ secrets.MCD_TOKEN }}
    fail-on: fail_only
```

## Outputs

| Output              | Description                                |
| ------------------- | ------------------------------------------ |
| `conclusion`        | `pass`, `warn`, or `fail`                  |
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
5. Displays the verdict and a per-asset explanation in the CI job output and step summary
6. The same explanation and a per-asset verdict table appear on the "MC Prevent CI Gate Result" check run on the PR
7. Raw API response available in a collapsible section

### Verdicts

MC Prevent returns one of three verdicts based on the risk assessment:

| Verdict  | What it means                      | `warn_and_fail` | `fail_only` | `none` | Check run on PR |
| -------- | ---------------------------------- | --------------- | ----------- | ------ | --------------- |
| **pass** | No significant risk detected       | Green           | Green       | Green  | Green           |
| **warn** | Risk detected — review recommended | Red             | Green       | Green  | Grey (neutral)  |
| **fail** | High risk — merge not recommended  | Red             | Red         | Green  | Red             |

**Note on CI job vs check run:** The CI job can only show green or red. The "MC Prevent CI Gate Result" check run posted on the PR shows the actual three-way severity (green/grey/red) along with a per-asset breakdown table showing each asset's verdict and reason. If you configure branch protection, require the check run (not the CI job) for accurate gating.

**Note on `fail-on` default:** When `fail-on` is not set in the workflow, the backend uses the setting configured in the Monte Carlo UI. This means teams can change the policy from the UI without modifying CI configuration.

### How the verdict is calculated

MC Prevent receives a risk assessment from the MC PR Agent for each data asset affected by the PR. It evaluates each asset against a decision matrix and takes the worst verdict across all assets.

**Decision matrix** — rules are evaluated top-to-bottom, first match wins:

| #   | Condition                                                  | Verdict  |
| --- | ---------------------------------------------------------- | -------- |
| 1   | Breaking change AND downstream assets depend on it         | **fail** |
| 2   | Active alerts highly correlated with the change            | **fail** |
| 3   | Breaking change AND no downstream assets                   | **warn** |
| 4   | Active alerts exist but low/no correlation with the change | **warn** |
| 5   | Everything else                                            | **pass** |

**Signals used per asset** — provided by the PR agent:

| Signal              | Description                                                               |
| ------------------- | ------------------------------------------------------------------------- |
| `change_type`       | How the asset is affected: `breaking` or `additive`                       |
| `alert_correlation` | Whether active alerts are related to the change: `high`, `low`, or `none` |
| `active_alerts`     | Number of unresolved alerts on the asset                                  |
| `downstream_assets` | Downstream assets (dashboards, tables) that depend on this asset          |

**Multi-asset PRs:** When a PR affects multiple data assets, each is evaluated independently. The final verdict is the **worst** across all assets — if one asset is `fail` and another is `pass`, the PR verdict is `fail`.

### Understanding the verdict explanation

The CI job output and the check run on the PR both include a per-asset explanation of why the verdict was reached. For each asset that triggered a warn or fail, the explanation describes the change type, downstream exposure, and what to verify. Assets that passed are summarized with their change type and downstream count.

The explanation ends with a sentence justifying the overall conclusion — for example, "Because the breaking change does not affect downstream assets, the conclusion is warn." This helps you understand why a high-risk PR might get warn instead of fail, or vice versa.

### What `fail-on` controls

| Setting         | Behavior                                                                                                                                                              |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| _(unset)_       | Uses the setting from the Monte Carlo UI (Settings → Integrations → CI Gate). This is the default.                                                                    |
| `warn_and_fail` | Both warn and fail cause the step to exit non-zero (red). This draws attention to all risks.                                                                          |
| `fail_only`     | Only fail exits non-zero. Warnings are visible in the step summary and check run but don't break your pipeline. Good for teams that want to focus on critical issues. |
| `none`          | The step always passes (green). The verdict is only visible in the job output, step summary, and the check run on the PR. Use this for a silent, non-blocking setup.  |

### Behavior by setup stage

MC Prevent is designed for progressive adoption — it never blocks your CI due to incomplete setup. You can add the workflow file first and configure the remaining pieces at your own pace.

| Setup stage                                  | CI result          | What you'll see                                                         |
| -------------------------------------------- | ------------------ | ----------------------------------------------------------------------- |
| Workflow added, secrets not yet configured   | Pass (green)       | Step skips instantly — no API call is made                              |
| Secrets configured, PR agent not yet enabled | Pass (green)       | Step polls for up to `max-wait` seconds, then passes with no assessment |
| Secrets configured, PR agent enabled         | Pass / Warn / Fail | Full risk verdict based on the PR agent's analysis                      |

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
Check the step output — if the verdict is `warn` (not `fail`), this is expected when `fail-on` is set to `warn_and_fail` (either explicitly or via the Monte Carlo UI). The check run on the PR will show grey (neutral), not red. Set `fail-on: fail_only` to only block on fail verdicts, or `fail-on: none` if you don't want any verdict to fail the CI job. You can also change this in the Monte Carlo UI under Settings → Integrations → CI Gate.

## Resources

- [Monte Carlo Documentation](https://docs.getmontecarlo.com)
- [MC Prevent Overview](https://docs.getmontecarlo.com/docs/github)
- [Source Code](https://github.com/monte-carlo-data/mc-prevent-action)
