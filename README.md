# rerun-action

A GitHub Action that reruns failed jobs, optionally only when specific patterns are found in logs.

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
      - uses: k1LoW/rerun-action@4bb68c6192bf65d175fbad2ebe27b8504b3a65c1 # v1.0.0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          pattern: 'Network partition'
          job: 'build-and-test'
          max_attempts: 2
```

### Rerun with multiple patterns (matches any)

```yaml
      - uses: k1LoW/rerun-action@4bb68c6192bf65d175fbad2ebe27b8504b3a65c1 # v1.0.0
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
      - uses: k1LoW/rerun-action@4bb68c6192bf65d175fbad2ebe27b8504b3a65c1 # v1.0.0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          pattern: 'Network partition'
          job: |
            build-and-test
            integration-test
```

### Rerun when a pattern is found across all jobs

```yaml
      - uses: k1LoW/rerun-action@4bb68c6192bf65d175fbad2ebe27b8504b3a65c1 # v1.0.0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          pattern: 'Network partition'
```

### Rerun only if a specific job failed (no log check)

```yaml
      - uses: k1LoW/rerun-action@4bb68c6192bf65d175fbad2ebe27b8504b3a65c1 # v1.0.0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          job: 'build-and-test'
```

### Rerun unconditionally on any failure

```yaml
      - uses: k1LoW/rerun-action@4bb68c6192bf65d175fbad2ebe27b8504b3a65c1 # v1.0.0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          max_attempts: 3
```

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `run_id` | Yes | | The workflow run ID to rerun |
| `pattern` | No | | Pattern to search for in logs (newline-separated for multiple). If omitted, rerun unconditionally |
| `job` | No | | Job name(s) to check (newline-separated for multiple). Narrows log search scope when used with `pattern`. When used alone, reruns only if any specified job failed |
| `github_token` | No | `${{ github.token }}` | GitHub Token with `actions:write` permission |
| `max_attempts` | No | `3` | Maximum run attempts to allow retry |
