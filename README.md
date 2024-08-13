# Matrix Output

Collect outputs from each matrix job. Currently, setting output for matrix jobs will cause outputs of earlier completed jobs to be overwritten by jobs completed later (see [discussion](https://github.com/orgs/community/discussions/26639)). This action allows the output from each matrix job to be collected into a JSON list to be utilized by dependent jobs.

The `matrix-output` job is intended to only be used once per job. Attempting to utilize this action multiple times within the same job will cause the second use of this action to fail.

## Examples

```yaml
# CI.yaml
jobs:
  build:
    name: Build ${{ matrix.build.name }}
    # These permissions are needed to:
    # - Use `matrix-output`: https://github.com/beacon-biosignals/matrix-output#permissions
    permissions:
      actions: read
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        build:
          - name: App One
            repo: user/app1
          - name: App Two
            repo: user/app2
    outputs:
      json: ${{ steps.matrix-output.outputs.json }}
    steps:
      - uses: docker/build-push-action@v6
        id: build-push
        with:
          tags: ${{ matrix.build.repo }}:latest
          push: true
      - uses: beacon-biosignals/matrix-output@v1
        id: matrix-output
        with:
          yaml: |
            name: ${{ matrix.build.name }}
            image: ${{ matrix.build.repo }}@${{ steps.build-push.outputs.digest }}

  test:
    name: Test ${{ matrix.build.name }}
    needs: build
    runs-on: ubuntu-latest
    container:
      image: ${{ matrix.build.image }}
    strategy:
      fail-fast: false
      matrix:
        build: ${{ fromJSON(needs.build.outputs.json) }}
    steps:
      ...
```

## Inputs

The `matrix-output` action supports the following inputs:

| Name             | Description | Required | Example |
|:-----------------|:------------|:---------|:--------|
| `yaml`           | A string representing a YAML data. Typically, a simple dictionary of key/value pairs. | Yes | <pre><code class="language-yaml">name: ${{ matrix.name }}&#10;...</code></pre> |

## Outputs

| Name   | Description | Example |
|:-------|:------------|:--------|
| `json` | A string representing a JSON list of dictionaries. Each dictionary in the list contains the output for a single job from the job matrix. The order of this list corresponds to the job index (i.e. `strategy.job-index`). | <pre><code class="language-json">[&#10;  {&#10;    "name": "Server.jl",&#10;    ...&#10;  },&#10;  {&#10;    "name": "Client.jl",&#10;    ...&#10;  }&#10;]</code></pre> |

## Permissions

The follow [job permissions](https://docs.github.com/en/actions/using-jobs/assigning-permissions-to-jobs) are required to run this action:

```yaml
permissions:
  action: read
```
