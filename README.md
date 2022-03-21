# GitHub Action: skip build if pull_request "synchronize" does not update "paths"
Often GitHub Pull Requests have a longer life-time, where more commits are while the PR already exists.
All of these pushes trigger the "synchronize" event as the PR is updated to include the latest commits from the head branch.
When combined with the "paths" or "paths-ignore" trigger filters, this sometimes leads to extra builds,
when the last few commits do not change any of the filtered paths: as long as the full PR changes any
of the matching files, even commits that do not change anything will trigger the build.

This action can be used to filter those events.

- It reads the Action yml file, and parses the triggers to see if the action has "paths" or "paths-ignore" attached.
- It computes the diff from the synchronize event 
  ```javascript
  "action": "synchronize",
  "after": "abcdefg", // most recent commit of this push
  "before": "1234567", // latest commit prior to this push
  ```
- If the diff does not edit any of the paths, or only ignored paths, the output `skip` will be defined.
- You can then abort the rest of the pipeline!

## Example usage with output

```yaml
name: build
on:
  workflow_dispatch: {}
  pull_request:
    paths:
      - 'app/**'
      - 'shared-models/**'

jobs:
  shouldskip:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
      - id: shouldskip
        uses: synchronize-skip-action@v1
    outputs:
      skip: ${{ steps.shouldskip.skip }}

  build:
    needs: shouldskip
    if: ${{ needs.shouldskip.skip == "skip" }}
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps: [] # more here
```

## Example usage with failure mode
- Upside: simpler
- Downside: same job so possibly requires an expensive runner (macOS)

```yaml
name: build
on:
  workflow_dispatch: {}
  pull_request:
    paths:
      - 'app/**'
      - 'shared-models/**'

jobs:
  build:
    steps:
    - name: Checkout
      uses: actions/checkout@v2
    - id: shouldskip
      uses: synchronize-skip-action@v1
      with: { fail: true }
    - id: info
      if: ${{ steps.shouldskip.outcomes == 'failure' }}
      run: echo aborted
    - # more steps here
```
