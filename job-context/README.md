# Job Context

Provides additional context for the currently running job. GitHub Actions provides a [`job` context](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/accessing-contextual-information-about-workflow-runs#job-context) but is missing some pieces including the job name and the job ID (the numeric value as used by the GitHub API).

## Requirements

The `job-context` action requires that the job in which this action is used has a job name that is unique within the workflow file. By default job names are unique so this is only a problem if you specify a custom `jobs.<job_key>.name`. If the job name is not unique within the workflow this action will fail and report the ambiguous job name.

Additionally, this job currently does not support job names which utilize GHA expressions using the contexts: `needs`, `vars`, or `inputs`.

## Examples

```yaml
# CI.yaml
jobs:
  demo:
    name: Demo
    # These permissions are needed to:
    # - Use `job-context`: https://github.com/beacon-biosignals/job-context#permissions
    permissions:
      context: read
      actions: read
    runs-on: ubuntu-latest
    strategy:
      matrix:
        version:
          - "1.0"
          - "2.0"
    steps:
      - uses: beacon-biosignals/job-context@v1
        id: job
      - run: |
          echo "job-name=${{ steps.job.outputs.name }}  # e.g. Demo (1.0)
          echo "job-id=${{ steps.job.outputs.id }}      # e.g. 28842064821
```

## Inputs

The `job-context` action supports the following inputs:

| Name             | Description | Required | Example |
|:-----------------|:------------|:---------|:--------|
| `path`           | The path to the cloned repo containing the workflow. Should only be needed by other GHAs using this action. | No | <pre><code>${{ github.action_path }}/repo</code></pre> |

## Outputs

| Name   | Description | Example |
|:-------|:------------|:--------|
| `name` | The rendered job name. | <pre><code>Demo (1.0)</code></pre> |
| `id`   | The numeric job ID as used by the GitHub API. Not be be confused with the workflow job key `github.job`. | <pre><code>28842064821</code></pre> |

## Permissions

The follow [job permissions](https://docs.github.com/en/actions/using-jobs/assigning-permissions-to-jobs) are required to run this action:

```yaml
permissions:
  actions: read
  context: read
```
