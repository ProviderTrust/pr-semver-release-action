# pr-semver-release-action
An opinionated GitHub action that makes it easy to create semver releases based on merging labeled PRs.

## Usage
Before using this action, you should manually create a tag and release in your repository representing the initial
version. This action requires that release/tag to exist in order to calculate the first release version that it creates.
This version tag must be a semantic version and have a prefix that matches the `version_prefix` input. For example, if
`version_prefix` is set to `v`, the initial version tag should be something like `v0.0.0`.

Then just create a workflow in your repository like this:
```yaml
name: Release or Label Workflow
run-name: ${{ github.event_name == 'push' && 'Create Release Version' || 'Require PR Labels' }}

on:
  pull_request:
    types: [opened, labeled, unlabeled, synchronize]
  push:
    branches:
      - main

jobs:
  release_or_label:
    name: ${{ github.event_name == 'push' && 'release_on_push' || 'require_label' }}
    runs-on: ubuntu-latest
    steps:
      - uses: 7Factor/pr-semver-release-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```
