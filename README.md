# rerun-action

A GitHub Action that reruns failed jobs, optionally only when specific patterns are found in logs.

## Usage

This action is designed to be used with the `workflow_run` event so that the target run has already completed before the rerun is triggered.

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
      actions: write
    steps:
      - uses: k1LoW/rerun-action@v0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          pattern: 'Network partition'
          job: 'build-and-test'
          github_token: ${{ secrets.GITHUB_TOKEN }}
          max_attempts: '2'
```

### Rerun with multiple patterns (matches any)

```yaml
      - uses: k1LoW/rerun-action@v0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          pattern: |
            Network partition
            Connection reset
            Timeout exceeded
          job: 'build-and-test'
          github_token: ${{ secrets.GITHUB_TOKEN }}
          max_attempts: '3'
```

### Rerun when a pattern is found across all jobs

```yaml
      - uses: k1LoW/rerun-action@v0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          pattern: 'Network partition'
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

### Rerun only if a specific job failed (no log check)

```yaml
      - uses: k1LoW/rerun-action@v0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          job: 'build-and-test'
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

### Rerun unconditionally on any failure

```yaml
      - uses: k1LoW/rerun-action@v0
        with:
          run_id: ${{ github.event.workflow_run.id }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          max_attempts: '3'
```

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `run_id` | Yes | | The workflow run ID to rerun |
| `pattern` | No | | Pattern to search for in logs (newline-separated for multiple). If omitted, rerun unconditionally |
| `job` | No | | Job name to check. Narrows log search scope when used with `pattern`. When used alone, reruns only if this job failed |
| `github_token` | Yes | | GitHub Token with `actions:write` permission |
| `max_attempts` | No | `3` | Maximum run attempts to allow retry |
