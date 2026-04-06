# rerun-action

A GitHub Action that reruns failed jobs, optionally only when specific patterns are found in logs.

This repository also provides `k1LoW/rerun-action/check`, a companion action that inspects a running job log and returns whether it matches a pattern.

## How it works

This action uses `gh run rerun <run_id> --failed` to rerun **all failed jobs** in a workflow run. The `job` and `pattern` inputs control **whether** to trigger a rerun, not **which** jobs get rerun.

- `job` and `pattern` are conditions for deciding whether to rerun
- When a rerun is triggered, all failed jobs in the run (and their dependents) are rerun
- The rerun creates a new attempt of the same workflow run

This action must be used with the `workflow_run` event. Since `gh run rerun` requires the target run to be completed, you cannot rerun a run from within itself. The `workflow_run` event fires after the target workflow finishes, making it the right trigger for this action.

## Usage

### Rerun when a specific pattern is found in a specific job's log

```yaml
name: Retry on transient failure

on:
  workflow_run:
    workflows: ["CI"]
    types:
      - completed

jobs:
  retry:
    runs-on: ubuntu-slim
    if: ${{ github.event.workflow_run.conclusion == 'failure' }}
    permissions:
      contents: read
      actions: write
    steps:
      - uses: k1LoW/rerun-action@80c58dc1309b82e2746743ea0dc3f726313412a1 # v1.2.0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          pattern: 'Network partition'
          job: 'build-and-test'
          max_attempts: 2
```

### Rerun with multiple patterns (matches any)

```yaml
      - uses: k1LoW/rerun-action@80c58dc1309b82e2746743ea0dc3f726313412a1 # v1.2.0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          pattern: |
            Network partition
            Connection reset
            Timeout exceeded
          job: 'build-and-test'
          max_attempts: 3
```

### Rerun with multiple jobs

When multiple jobs are specified, a rerun is triggered if **any** of them failed. Logs from all failed jobs are searched when `pattern` is also specified.

```yaml
      - uses: k1LoW/rerun-action@80c58dc1309b82e2746743ea0dc3f726313412a1 # v1.2.0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          pattern: 'Network partition'
          job: |
            build-and-test
            integration-test
```

### Rerun when a pattern is found across all jobs

```yaml
      - uses: k1LoW/rerun-action@80c58dc1309b82e2746743ea0dc3f726313412a1 # v1.2.0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          pattern: 'Network partition'
```

### Rerun only if a specific job failed (no log check)

```yaml
      - uses: k1LoW/rerun-action@80c58dc1309b82e2746743ea0dc3f726313412a1 # v1.2.0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          job: 'build-and-test'
```

### Rerun unconditionally on any failure

```yaml
      - uses: k1LoW/rerun-action@80c58dc1309b82e2746743ea0dc3f726313412a1 # v1.2.0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          max_attempts: 3
```

### Check whether a rerun was triggered

```yaml
      - id: rerun
        uses: k1LoW/rerun-action@80c58dc1309b82e2746743ea0dc3f726313412a1 # v1.2.0
        with:
          run_id: ${{ github.event.workflow_run.id }}

      - run: echo "reran=${{ steps.rerun.outputs.reran }}"
```

### Notify only when a rerun was triggered

This example uses a pseudo notification action to show how to branch on `steps.rerun.outputs.reran`.

```yaml
      - id: rerun
        uses: k1LoW/rerun-action@80c58dc1309b82e2746743ea0dc3f726313412a1 # v1.2.0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          pattern: 'Network partition'
          job: 'build-and-test'

      - uses: example/notify-action@v1
        if: ${{ steps.rerun.outputs.reran == 'true' }}
        with:
          message: "Rerun triggered for workflow run ${{ github.event.workflow_run.id }}"
```

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `run_id` | Yes | | The workflow run ID to rerun |
| `pattern` | No | | Pattern to search for in logs (newline-separated for multiple). If omitted, rerun unconditionally |
| `job` | No | | Job name(s) to check (newline-separated for multiple). Narrows log search scope when used with `pattern`. When used alone, reruns only if any specified job failed |
| `github_token` | No | `${{ github.token }}` | GitHub Token with `actions:write` permission |
| `max_attempts` | No | `3` | Maximum run attempts to allow retry |

## Outputs

| Name | Description |
|------|-------------|
| `reran` | `true` if this action triggered `gh run rerun`, otherwise `false` |

## `check` action

`k1LoW/rerun-action/check` is intended for use inside the target job itself. It does not rerun anything. Instead, it reads the current run's job log, retries a few times while the log is being flushed, and returns whether the specified pattern was found.

### Usage

```yaml
jobs:
  build-and-test:
    name: build-and-test
    runs-on: ubuntu-slim
    steps:
      - name: Run tests
        run: make test

      - id: check
        if: ${{ always() }}
        uses: k1LoW/rerun-action/check@main
        with:
          job: build-and-test
          pattern: Network partition

      - uses: example/notify-action@v1
        if: ${{ steps.check.outputs.matched == 'true' }}
        with:
          message: "This job matched the retry-worthy pattern"
```

### Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `job` | Yes | | Job name to inspect |
| `pattern` | Yes | | Pattern to search for in the job log |
| `github_token` | No | `${{ github.token }}` | GitHub Token with `actions:read` permission |
| `retry_count` | No | `5` | Number of retries while waiting for log output to become available |
| `retry_interval` | No | `3` | Seconds to wait between log fetch retries |

### Outputs

| Name | Description |
|------|-------------|
| `matched` | `true` if the pattern was found in the job log, otherwise `false` |
| `job_id` | Database ID of the inspected job |
| `reason` | Result summary such as `matched`, `pattern_not_found`, or `log_unavailable` |
