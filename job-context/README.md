# Job Context

Provides additional context for the currently running job. GitHub Actions provides a [`job` context](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/accessing-contextual-information-about-workflow-runs#job-context) but is missing some pieces including the job name and the job ID (the numeric value as used by the GitHub API).

## Examples

```yaml
# CI.yaml
jobs:
  build:
    name: Build
    # These permissions are needed to:
    # - Use `job-context`: https://github.com/beacon-biosignals/job-context#permissions
    permissions:
      context: read
      actions: read
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        version:
          - "1.0"
          - "2.0"
    steps:
      - uses: beacon-biosignals/job-context@v1
        id: job
      - run: |
          echo "job-name=${{ steps.job.outputs.name }}  # e.g. Build (1.0)
          echo "job-id=${{ steps.job.outputs.id }}      # e.g. 28842064821
```

## Inputs

The `job-context` action supports the following inputs:

| Name             | Description | Required | Example |
|:-----------------|:------------|:---------|:--------|
| `path`           | The workflow repo path | No | <pre><code>${{ github.action_path }}/repo</code></pre> |

## Outputs

| Name   | Description | Example |
|:-------|:------------|:--------|
| `name` | The rendered job name. | <pre><code>Build (1.0)</code></pre> |
| `id`   | The numeric job ID as used by the GitHub API. Not be be confused with `${{ github.job }}`. | <pre><code>28842064821</code></pre> |

## Permissions

The follow [job permissions](https://docs.github.com/en/actions/using-jobs/assigning-permissions-to-jobs) are required to run this action:

```yaml
permissions:
  actions: read
  context: read
```
