# GitHub Metrics — Setup & Maintenance

This document describes how the profile metrics workflow is configured, how generated files are published, and how to troubleshoot common failures.

Repository: [AmirSenpai/AmirSenpai](https://github.com/AmirSenpai/AmirSenpai)  
Workflow: [`.github/workflows/main.yml`](.github/workflows/main.yml)  
Generator: [lowlighter/metrics](https://github.com/lowlighter/metrics)

## 1. What the workflow does

The GitHub Actions workflow runs the official `lowlighter/metrics` action and generates an SVG profile dashboard.

- Runs automatically once every hour.
- Can also be started manually from **Actions → Metrics → Run workflow**.
- Runs on pushes to `main` or `master`.
- Uses the `classic` template.
- Writes the generated file to `github-metrics.svg` in the repository root.
- Commits the generated file to `main`.
- Displays `github-metrics.svg` in the profile `README.md`, so visitors see the dashboard directly on the GitHub profile.

The schedule is `0 * * * *`: once per hour, at minute 0. GitHub Actions cron schedules use UTC. The `config_timezone: Asia/Tehran` setting changes dates displayed inside the metrics; it does not change the schedule timezone.

## 2. Output action: how publishing works

Metrics supports several output methods. This repository uses **`output_action: commit`**.

| Option | Purpose |
| --- | --- |
| `commit` | Commit the generated output directly to the target branch. **Used here.** |
| `pull-request` | Push output to a new branch and open a pull request. |
| `pull-request-merge` | Open a pull request and merge it. |
| `pull-request-squash` | Open a pull request and squash-merge it. |
| `pull-request-rebase` | Open a pull request and rebase-merge it. |
| `gist` | Publish the output to an existing GitHub Gist. |
| `none` | Leave generated files in the runner for custom handling. |

### Permissions required for the current setup

The workflow grants:

```yaml
permissions:
  contents: write
```

This allows the action's commit step to write generated files to this repository.

The publishing token is configured explicitly:

```yaml
output_action: commit
committer_token: ${{ github.token }}
committer_branch: main
committer_message: "chore: update GitHub metrics [Skip GitHub Action]"
```

- `github.token` is the short-lived GitHub Actions token for this repository.
- `committer_branch` selects the destination branch.
- `committer_message` describes the automated commit.
- The `[Skip GitHub Action]` marker prevents the generated commit from starting another workflow run.

The repository's **Settings → Actions → General → Workflow permissions** must allow read and write permissions for the workflow token. If repository policy restricts the token, the commit step may fail even when the YAML permissions are present.

### Data token vs. publishing token

These two tokens have different jobs:

- `token: ${{ secrets.METRICS_TOKEN }}` is used to read GitHub data for the dashboard.
- `committer_token: ${{ github.token }}` is used to publish the generated SVG.

The `METRICS_TOKEN` secret must exist under **Repository → Settings → Secrets and variables → Actions**. Keep it secret; never paste it into this document or commit it to the repository.

The Metrics documentation says no PAT scopes are required by default, although some optional plugins may need additional scopes.

## 3. Current dashboard configuration

The workflow currently enables:

- Profile header, activity, community, repositories, and metadata.
- Contribution calendar and half-year isometric calendar.
- Most-used programming and markup languages.
- Stargazer charts and recent star history.
- Twemoji rendering and the Asia/Tehran display timezone.

### Why the Lines plugin is disabled

The Lines plugin was disabled after the latest failed run.

The run log showed:

```text
TypeError: Cannot read properties of null (reading 'login')
at source/plugins/lines/index.mjs
```

The plugin then left incomplete data for the Classic template, which caused a second rendering error involving `sections.includes`.

This is a plugin data-processing failure, not a YAML parsing or output-permission failure. The Lines plugin is intentionally omitted until the underlying Metrics issue or offending repository data can be identified. The rest of the dashboard remains enabled.

## 4. How to run or verify it

1. Open the repository's **Actions** tab.
2. Select **Metrics**.
3. Use **Run workflow** to start a manual run.
4. Open the newest run and inspect the `github-metrics` job.
5. When it succeeds, check that `github-metrics.svg` has a recent commit on `main`.

A successful render followed by a failed commit usually points to repository workflow permissions or token policy. A failed render should be diagnosed from the first error in the action log, before later template errors.

## 5. Troubleshooting

### The workflow does not start

- Confirm Actions are enabled for the repository.
- Check that the workflow file is under `.github/workflows/`.
- Remember that scheduled runs use UTC and may be delayed by GitHub.
- Use **Run workflow** to test without waiting for the next hourly schedule.

### Metrics cannot read GitHub data

- Confirm the `METRICS_TOKEN` Actions secret exists and has not expired or been revoked.
- Check the first API or authentication error in the run log.
- Only add PAT scopes when a selected plugin explicitly requires them.

### The generated SVG cannot be committed

- Confirm `contents: write` is present at workflow or job level.
- Confirm repository workflow permissions allow read and write access.
- Confirm `committer_branch` points to an existing branch.
- Check the log for branch protection or token permission errors.

### Rendering fails

- Find the first plugin error in the run log.
- Do not assume a later template error is the root cause; it may be a consequence of a plugin failing earlier.
- Temporarily disable the failing optional plugin, then re-enable it after its issue is resolved.

## 6. Official references

- [Metrics core configuration](https://github.com/lowlighter/metrics/blob/master/source/plugins/core/README.md)
- [Configuring output action](https://github.com/lowlighter/metrics/blob/master/source/plugins/core/README.md#-configuring-output-action)
- [Lines plugin options](https://github.com/lowlighter/metrics/blob/master/source/plugins/lines/README.md)
- [GitHub Actions token permissions](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token)
- [GitHub Actions workflow syntax and schedule](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions#onschedule)
