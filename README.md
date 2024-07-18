# Auto Merge wrapper

A simple composite GitHub Action that composes the Actions needed to automatically merge a Pull Requests when all required checks have passed (e.g. CI, review, etc).

## Usage

### On Pull Request

```yaml
name: Gather Pull Request Metadata
on:
  pull_request:
    types: [ opened, reopened, synchronize ]
    branches: [ master ]

permissions:
  contents: read

jobs:
  gather-metadata:
    if: github.repository_owner == 'sclorg'
    runs-on: ubuntu-latest

    steps:
      - name: Repository checkout
        uses: actions/checkout@v4

      - id: Metadata
        name: Gather Pull Request Metadata
        uses: redhat-plumbers-in-action/gather-pull-request-metadata@v1

      - name: Upload artifact with gathered metadata
        uses: actions/upload-artifact@v4
        with:
          name: pr-metadata
          path: ${{ steps.Metadata.outputs.metadata-file }}
```

```yaml
name: Auto Merge
on:
  workflow_run:
    workflows: [ Gather Pull Request Metadata ]
    types:
      - completed

permissions:
  contents: read

jobs:
  download-metadata:
    if: >
      github.event.workflow_run.event == 'pull_request' &&
      github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest

    outputs:
      pr-metadata: ${{ steps.Artifact.outputs.pr-metadata-json }}

    steps:
      - id: Artifact
        name: Download Artifact
        uses: redhat-plumbers-in-action/download-artifact@v1
        with:
          name: pr-metadata

  auto-merge:
    needs: [ download-metadata ]
    runs-on: ubuntu-latest

    permissions:
      # required for merging PRs
      contents: write
      # required for PR comments and setting labels
      pull-requests: write

    steps:
      - name: Auto Merge wrapper
        uses: sclorg/auto-merge-wrapper@v1
        with:
          pr-metadata: ${{ needs.download-metadata.outputs.pr-metadata }}
          token: ${{ secrets.GITHUB_TOKEN }}
```

### On Schedule / On Demand

```yaml
name: Auto Merge Scheduled / On Demand
on:
  schedule:
    # Workflow runs every 30 minutes
    - cron: '*/30 * * * *'
  workflow_dispatch:
    inputs:
      pr-number:
        description: 'Pull Request number/s ; when not provided, the workflow will run for all open PRs'
        required: true
        default: '0'

permissions:
  contents: read

jobs:
  # Get all open PRs
  gather-pull-requests:
    if: github.repository_owner == 'sclorg'
    runs-on: ubuntu-latest

    outputs:
      pr-numbers: ${{ steps.get-pr-numbers.outputs.result }}
      pr-numbers-manual: ${{ steps.parse-manual-input.outputs.result }}

    steps:
      - id: get-pr-numbers
        if: inputs.pr-number == '0'
        name: Get all open PRs
        uses: actions/github-script@v7
        with:
          # !FIXME: this is not working if there is more than 100 PRs opened
          script: |
            const { data: pullRequests } = await github.rest.pulls.list({
              owner: context.repo.owner,
              repo: context.repo.repo,
              state: 'open',
              per_page: 100
            });
            return pullRequests.map(pr => pr.number);

      - id: parse-manual-input
        if: inputs.pr-number != '0'
        name: Parse manual input
        run: |
          # shellcheck disable=SC2086
          echo "result="[ ${{ inputs.pr-number }} ]"" >> $GITHUB_OUTPUT
        shell: bash

  validate-pr:
    name: 'Validation of Pull Request #${{ matrix.pr-number }}'
    needs: [ gather-pull-requests ]
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        pr-number: ${{ inputs.pr-number == 0 && fromJSON(needs.gather-pull-requests.outputs.pr-numbers) || fromJSON(needs.gather-pull-requests.outputs.pr-numbers-manual) }}

    permissions:
      # required for merging PRs
      contents: write
      # required for PR comments and setting labels
      pull-requests: write

    steps:
      - name: Auto Merge wrapper
        uses: sclorg/auto-merge-wrapper@v1
        with:
          pr-number: ${{ matrix.pr-number }}
          token: ${{ secrets.GITHUB_TOKEN }}
```
