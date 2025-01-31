# pr-semver-release-action
An opinionated GitHub action that makes it easy to create semver releases based on merging labeled PRs.

## Usage
Before using this action, you should manually create a tag and release in your repository representing the initial
version. This action requires that release/tag to exist in order to calculate the first release version that it creates.
This version tag must be a semantic version and have a prefix that matches the `version_prefix` input. For example, if
`version_prefix` is set to `v`, the initial version tag should be something like `v0.0.0`.

### Single Workflow
The following example workflow can be used when there are no additional steps needed after the release is created, such
as building and uploading artifacts to the release. This is a good option for publishing Terraform modules or GitHub
Actions that do not need to be compiled.

Just create a workflow in your repository like this:
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
          update_major_minor_tags: true # Optional, but recommended for Terraform modules and GitHub Actions
```

### Separate Workflows
Often times, you may need to perform additional steps after the release is created, such as building and uploading
artifacts to the release. In this case, you can create two separate workflows: one for requiring PR labels and one for
creating the release.

#### Require PR Labels Workflow
```yaml
name: Require PR Labels

on:
  pull_request:
    types: [opened, labeled, unlabeled, synchronize]

jobs:
  require_label:
    runs-on: ubuntu-latest
    steps:
      - uses: 7Factor/pr-semver-release-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

#### Create Release Workflow
```yaml
name: Release on Push

on:
  push:
    branches:
      - main

jobs:
  # Generally, the release should be created first, so that the release version is available to subsequent steps.
  # Additionally, if the release fails, the workflow can be retried without wasting time on other steps.
  create_release:
    runs-on: ubuntu-latest
    steps:
      - uses: 7Factor/pr-semver-release-action@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}

  build_artifact:
    needs: [create_release] # Or skip this line to run in parallel with create_release
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
    
      # ... Steps for building ...

      - uses: actions/upload-artifact@v4
        with:
          name: build-artifact
          path: ./dist # Upload the dist directory or w/e build artifacts you have

  upload_release_artifact:
    needs: [create_release, build_artifact]
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-artifact
          path: . # Don't specify the folder name you uploaded since that folder is part of the uploaded artifact

      - name: Upload Release Artifacts
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          zip -r artifact.zip dist/
          gh release upload ${{ needs.create_release.outputs.tag_name }} artifact.zip --repo $GITHUB_REPOSITORY

  # Deploy jobs could follow here
```
