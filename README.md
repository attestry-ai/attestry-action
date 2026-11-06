# Attestry Compliance Check — GitHub Action

A CI/CD compliance gate that checks your AI system against **EU AI Act**, **Colorado AI Act**, **NIST AI RMF**, and **ISO 42001** using the [Attestry](https://attestry.ai) API.

Block merges when compliance scores drop below your threshold, critical gaps are unresolved, **or a multi-approver workflow has not yet released the attestation** (Ship-Gate Integration — Prompt E.2).

## Quick Start

```yaml
# .github/workflows/compliance.yml
name: AI Compliance
on:
  pull_request:
    branches: [main]

permissions:
  pull-requests: write

jobs:
  compliance:
    runs-on: ubuntu-latest
    steps:
      - name: Attestry Compliance Gate
        uses: attestry-ai/attestry-action@v1
        with:
          api-key: ${{ secrets.ATTESTRY_API_KEY }}
          system-id: ${{ vars.AI_SYSTEM_ID }}
          min-score: '75'
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `api-key` | Yes | — | Attestry API key (starts with `rs_`). Store as a repository secret. |
| `system-id` | Yes | — | UUID of the AI system to check. |
| `min-score` | No | `70` | Minimum compliance score (0-100) to pass the gate. |
| `fail-on-gaps` | No | `true` | Fail the check if any **critical** compliance gaps exist, even if the score passes. |
| `fail-on-missing-assessment` | No | `true` | Fail if no completed assessment exists for the system. |
| `frameworks` | No | `''` | Comma-separated frameworks to check (e.g. `eu-ai-act,colorado-ai-act,nist-ai-rmf,iso-42001`). Empty = all. |
| `api-url` | No | `https://attestry.ai` | Override for self-hosted or staging environments. |
| `comment-on-pr` | No | `true` | Post a compliance summary comment on pull requests. |
| `attestation-id` | No | `''` | Attestation / build identifier for the **Ship-Gate Integration**. When set, the action additionally calls `POST /api/v1/ship-gate/check` and fails the build if a multi-approver workflow has not yet released the attestation. Leave empty to skip the ship-gate step. |

## Outputs

| Output | Description |
|--------|-------------|
| `gate` | Compliance gate: `pass` or `fail` |
| `score` | Compliance score (0-100), or `null` if no assessment |
| `gap-count` | Total unresolved compliance gaps |
| `critical-gaps` | Number of critical-priority gaps |
| `high-gaps` | Number of high-priority gaps |
| `assessment-id` | UUID of the assessment that was checked |
| `assessment-date` | ISO 8601 date the assessment was completed |
| `reason` | Human-readable explanation of the compliance result |
| `ship-gated` | Ship-gate result: `true` if build is blocked, `false` otherwise. Empty when `attestation-id` is unset. |
| `ship-gate-reason` | Stable code: `awaiting_approvers` \| `rejected` \| `timed_out` (empty when not gated) |
| `ship-gate-state` | Resolved state: `gated` \| `released` \| `rejected` \| `timed_out` |
| `approvers-pending` | Comma-separated user IDs still owed a decision before the gate releases |
| `execution-id` | Approval-chain execution ID backing the ship-gate |
| `chain-id` | Approval-chain template ID backing the execution |

## Features

- **Score gate** — fails the build if the compliance score is below `min-score`
- **Critical gap gate** — fails if any critical gaps exist (when `fail-on-gaps` is `true`)
- **Ship-gate** — fails the build when a multi-approver workflow has not yet released the attestation (opt-in via `attestation-id`)
- **PR comments** — posts (and updates) a compliance summary on pull requests
- **Job summary** — writes a formatted table to the GitHub Actions summary tab
- **Framework filtering** — check specific regulatory frameworks only
- **Structured logs** — formatted output for build log readability

## Usage Examples

### Basic — block PRs below score 70

```yaml
- uses: attestry-ai/attestry-action@v1
  with:
    api-key: ${{ secrets.ATTESTRY_API_KEY }}
    system-id: '550e8400-e29b-41d4-a716-446655440000'
```

### Strict — EU AI Act only, score 90, fail on any critical gap

```yaml
- uses: attestry-ai/attestry-action@v1
  with:
    api-key: ${{ secrets.ATTESTRY_API_KEY }}
    system-id: ${{ vars.AI_SYSTEM_ID }}
    min-score: '90'
    fail-on-gaps: 'true'
    frameworks: 'eu-ai-act'
```

### Non-blocking audit (continue on failure)

```yaml
- uses: attestry-ai/attestry-action@v1
  continue-on-error: true
  with:
    api-key: ${{ secrets.ATTESTRY_API_KEY }}
    system-id: ${{ vars.AI_SYSTEM_ID }}
    min-score: '70'
    fail-on-gaps: 'false'
```

### Use outputs in downstream steps

```yaml
- name: Compliance check
  id: attestry
  uses: attestry-ai/attestry-action@v1
  with:
    api-key: ${{ secrets.ATTESTRY_API_KEY }}
    system-id: ${{ vars.AI_SYSTEM_ID }}

- name: Notify on failure
  if: steps.attestry.outputs.gate == 'fail'
  run: |
    echo "Compliance failed with score ${{ steps.attestry.outputs.score }}"
    echo "${{ steps.attestry.outputs.critical-gaps }} critical gaps need attention"
```

### Multiple AI systems (matrix strategy)

```yaml
strategy:
  matrix:
    system:
      - { id: 'uuid-1', name: 'Recommender', min: '75' }
      - { id: 'uuid-2', name: 'Credit Model', min: '90' }
steps:
  - uses: attestry-ai/attestry-action@v1
    with:
      api-key: ${{ secrets.ATTESTRY_API_KEY }}
      system-id: ${{ matrix.system.id }}
      min-score: ${{ matrix.system.min }}
```

### Ship-Gate — block deploys until multi-approver workflow releases the attestation

```yaml
- uses: attestry-ai/attestry-action@v1
  with:
    api-key: ${{ secrets.ATTESTRY_API_KEY }}
    system-id: ${{ vars.AI_SYSTEM_ID }}
    # Setting attestation-id activates the ship-gate. The action calls
    # POST /api/v1/ship-gate/check and fails the build if an approval
    # chain is still in_progress (or rejected / timed_out).
    attestation-id: ${{ github.sha }}
```

When `attestation-id` is provided, the action additionally calls `POST /api/v1/ship-gate/check` after the compliance gate. If a `ship_gates` row exists for `(system-id, attestation-id)` and the linked approval-chain execution has not reached `approved`, the build fails with a structured error message. When no ship-gate exists for the tuple (the default), the step is permissive — the build proceeds.

Reasons returned by the ship-gate:

| `ship-gate-reason` | Meaning |
|---|---|
| `awaiting_approvers` | Approval chain still in progress; `approvers-pending` lists the user IDs |
| `rejected` | Approval chain rejected the attestation |
| `timed_out` | Approval chain SLA elapsed without enough approvals |
| _(empty)_ | Either no ship-gate exists for the tuple, or the chain has been released |

## PR Comment

When running on a pull request with `comment-on-pr: true` (the default), the action posts a summary comment:

> ## Attestry Compliance Gate — PASS
>
> | Field | Value |
> |-------|-------|
> | Score | **82** / 100 (min: 75) |
> | Gaps | 3 total (0 critical, 1 high) |
> | Result | **pass** |

Previous Attestry comments are automatically replaced to avoid clutter.

## Badge

Add a compliance badge to your README:

```markdown
![Attestry](https://img.shields.io/badge/Attestry-Compliant-brightgreen?logo=data:image/svg+xml;base64,...)
```

Or use a dynamic badge tied to your latest workflow run:

```markdown
![Compliance](https://github.com/YOUR_ORG/YOUR_REPO/actions/workflows/compliance.yml/badge.svg)
```

## Setup

1. **Get an API key** — sign up at [attestry.ai](https://attestry.ai) and create an API key with `read:assessments` and `read:systems` permissions.
2. **Add the secret** — go to your repository Settings > Secrets > Actions and add `ATTESTRY_API_KEY`.
3. **Find your system ID** — in the Attestry dashboard, go to AI Systems and copy the UUID.
4. **Add the workflow** — create `.github/workflows/compliance.yml` using the quick start example above.

## Requirements

- A Attestry account with at least one AI system registered
- A completed compliance assessment for that system
- An API key with `read:assessments` and `read:systems` permissions
- `jq` and `curl` (pre-installed on all GitHub-hosted runners)

## License

MIT
