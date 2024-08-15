# Design Details

The goal of the `matrix-output` GitHub Action is to provide a single action which can allow users
to collect output from each job in a job matrix and utilize that information as part of `needs.<job_key>.outputs` in dependent jobs.

In order to accomplish this goal we need do two things:

1. Preserve output from each job in the matrix
2. Ensure the last running job in the matrix has combined all job matrix outputs

## Preserving job matrix output

In order to preserve the output from each job in the matrix we utilize GitHub artifact to store each job's output. The artifact naming convention used here is `matrix-output-${{ github.job }}-${{ strategy.job-index}}`.

We include both the `github.job` (YAML job key) and the `strategy.job-index` (matrix job index) to avoid artifact name collisions within a run attempt. The job key allows this action to be used by multiple jobs within the same workflow and the job index ensures matrix jobs use distinct artifacts. Additionally, the use of the job index allows us to ensure consistent ordering of the output.

At this time we do not support using this action multiple times within the same workflow job. We could support this in the future by possibly including `github.action` (step ID).

> Note: The [GitHub Actions documentation states that the `github.job`](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/accessing-contextual-information-about-workflow-runs#github-context) is the "`job_id`" of the current job. The `github.job` specifically refers to the YAML key used for the job which is different from the numeric `job_id` used by the GitHub API. In order to disambiguate this term here I'll be referring to the `github.job` as the job key or `job_key` and the GitHub API job ID as `job_id`.

## Ensure the last matrix job has combined all job matrix outputs

When using the `outputs` field for a matrix job only results from the last completed job in the matrix are accessible. We need the action to ensure that the last completed job in the matrix returns the outputs from all jobs in the matrix. Guaranteeing this turns out to be quite complex.

Consider a scenario in which there are two jobs in a matrix job: A and B. Job A completes the `matrix-output` action first but encounters a long running post step. Job B executes `matrix-output` after A but completes before A. In this scenario the final job A output would not contain the outputs for B. To address this problem we need to have B wait for A to complete to ensure the output is complete. We now need to answer these questions:

- How do we know when a job in the matrix has complete set of outputs?
- How can we determine we are the last running job in the matrix?

### How do we know when a job in the matrix has complete set of outputs?

Initially, determining if a job in the matrix has a complete set of output seems trivial. We just need to download all the artifacts for run with the pattern `matrix-output-${{ github.job }}-*` and check that the number of artifacts equals the `strategy.job-total`. Unfortunately, re-runs complicate things as we cannot determine by the artifacts alone if an artifact was created on the current attempt or from a previous attempt.

Consider a scenario in which there are two jobs in a job matrix (X and Y) which create distinct output on each execution. On the first run attempt only X completed successfully. The user then triggers a re-run of all jobs. In this second run attempt Y reaches the `matrix-output` action before X. When Y checks for which output artifacts are available it sees the artifact for X from the first run attempt and the artifact for Y for the second run attempt. If Y were to finish last on the second attempt we would produce an incorrect set of outputs.

To solve this new problem we need to be able to identify which artifacts are associated with the current run attempt. The GitHub API for listing artifacts doesn't provide this information. However, the GitHub API for listing jobs does contain the information we need. All we need to do is to be able to associate the output artifacts we create with the specific `job_id` which created the artifact

> Aside: As GitHub enforces that each run attempt cannot occur concurrently we can determine the artifact run attempt by using the list artifacts and list jobs API. This however is not enough to determine the specific job which produced the artifact though.

If we include the `job_id` of the current job as part of the output artifact we can then utilize the GitHub API job list endpoint to determine if that artifact is part of the current run attempt. As the [GitHub provided contexts](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/accessing-contextual-information-about-workflow-runs) do not provide the numeric `job_id` used by the GitHub API we'll utilize the custom GHA `job-context` to determine the `job_id` for the current job.

So now we can use the GitHub API job list endpoint to determine the set of jobs executed for the latest attempt and the GHA `action/download` to determine the `job_id`s of the latest (possibly outdated) outputs for each job in the matrix. By performing an intersection of these job IDs we can identify the jobs for the latest attempt which have produced output. We refer to this job set as the "known matrix jobs". If the number of known matrix jobs is equal to `strategy.job-total` we know that the current running job has a complete set of outputs.

### How can we determine we are the last running job in the matrix?

We can partially determine if we have a complete set of outputs. If the number of outputs is less than the `strategy.job-total` we know there is another job in the matrix which has yet to upload its output as an artifact.

For jobs with a complete set of output we can utilize the GitHub API jobs endpoint to determine which jobs in the matrix are still running. The set of "known matrix jobs" (see previous section) for a job with a complete set of output will provide us with the job IDs for all jobs in the matrix. By polling the GitHub API job endpoint and waiting when we find a running job (besides ourself) we can ensure that only a job with a complete set of outputs is the last to complete.

However, consider a scenario where there are two jobs in a job matrix (E and F) where both have complete sets of outputs. If these jobs wait until all other siblings jobs have completed they will end up waiting indefinitely as E waits on F and F waits on E.

What we need here is some kind of synchronization primitive to determine which jobs are waiting. So for jobs that have complete sets of outputs we create another artifact with the convention `matrix-output-sync-${{ github.job }}-${{ strategy.job-index }}` which indicates that a job is waiting. If we update our waiting loop to treat waiting sibling jobs as complete we can can avoid the waiting deadlock. Note we don't actually know which of these waiting jobs will complete last but it doesn't matter as all waiting jobs will have a complete set of outputs.

One concern about using waiting within the `matrix-output` GHA is that we could end up artificially extending the duration of a waiting job. To minimize the waiting duration we recommend that the `matrix-output` GHA is used as the last step in the matrix job. By following this recommendation the waiting duration should be near zero since the waiting jobs each have complete set of outputs and therefore all jobs in the matrix must have reached this step.
