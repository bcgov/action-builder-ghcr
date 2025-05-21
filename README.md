<!-- Badges -->
[![Issues](https://img.shields.io/github/issues/bcgov/action-builder-ghcr)](/../../issues)
[![Pull Requests](https://img.shields.io/github/issues-pr/bcgov/action-builder-ghcr)](/../../pulls)
[![MIT License](https://img.shields.io/github/license/bcgov/action-builder-ghcr.svg)](/LICENSE)
[![Lifecycle](https://img.shields.io/badge/Lifecycle-Experimental-339999)](https://github.com/bcgov/repomountie/blob/master/doc/lifecycle-badges.md)

# Conditional Container Builder with Fallback

This action builds Docker/Podman containers conditionally using a set of directories.  If any files were changed matching that, then build a container.  If those files were not changed, retag an existing build.

This is useful in CI/CD pipelines where not every package/app needs to be rebuilt.

This tool is currently strongly opinionated and generates images with a rigid structure below.  This is intended to become more flexible in future.

Package name: `<organization>/<repository>/<package>:<tag>`

Pull with: `docker pull ghcr.io/<organization>/<repository>/<package>:<tag>` 

Only GitHub Container Registry (ghcr.io) is supported so far.

# Usage

```yaml
- uses: bcgov/action-builder-ghcr@vX.Y.X
  with:
    ### Required

    # Package name
    package: frontend


    ### Typical / recommended

    # Sets the build context/directory, which contains the build files
    # Optional, defaults to package name
    build_context: ./frontend

    # Sets the Dockerfile with path
    # Optional, defaults to {package}/Dockerfile or {build_context}/Dockerfile
    build_file: ./frontend/Dockerfile

    # Fallback tag, used if no build was generated
    # Optional, defaults to nothing, which forces a build
    # Non-matching or malformed tags are rejected, which also forced a build
    tag_fallback: test

    # Tags to apply to the image
    # Optional, defaults to pull request number
    tags: |
      pr123
      demo

    # Bash array to diff for build triggering
    # Optional, defaults to nothing, which forces a build
    triggers: ('frontend/' 'backend/' 'database/')


    ### Usually a bad idea / not recommended

    # Sets a list of [build-time variables](https://docs.docker.com/engine/reference/commandline/buildx_build/#build-arg)
    # Optional, defaults to sample content
    build_args: |
      ENV=build

    # Overrides the default branch to diff against
    # Defaults to the default branch, usually `main`
    diff_branch: ${{ github.event.repository.default_branch }}

    # Repository to clone and process
    # Useful for consuming other repos, like in testing
    # Defaults to the current one
    repository: ${{ github.repository }}

    # Specify token (GH or PAT), instead of inheriting one from the calling workflow
    token: ${{ secrets.GITHUB_TOKEN }}


    ### Deprecated

    # Single-value tag input has been deprecated and will be removed in a future release
    # Please use inputs.tags, which can handle multiple values
    tag: do not use!

```

# Example, Single Build

Build a single subfolder with a Dockerfile in it.  Deletes old packages, keeping the last 50.  Runs on pull requests (PRs).

Create or modify a GitHub workflow, like below.  E.g. `.github/workflows/pr-open.yml`

```yaml
name: Pull Request

on:
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  builds:
    permissions:
      packages: write
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Builds
        uses: bcgov/action-builder-ghcr@vX.Y.Z
        with:
          package: frontend
          tag_fallback: test
          triggers: ('frontend/')
```

# Example, Single Build with build_context, build_file and multiple tags

Same as previous, but specifying build folder and Dockerfile.

Create or modify a GitHub workflow, like below.  E.g. `.github/workflows/pr-open.yml`

```yaml
name: Pull Request

on:
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  builds:
    permissions:
      packages: write
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Builds
        uses: bcgov/action-builder-ghcr@vX.Y.Z
        with:
          package: frontend
          build_context: ./
          build_file: subdir/Dockerfile
          tags: |
            ${{ github.event.number }}
            ${{ github.sha }}
            latest
          tag_fallback: test
          token: ${{ secrets.GITHUB_TOKEN }}
          triggers: ('frontend/')
```

# Example, Matrix Build

Build from multiple subfolders with Dockerfile in them.  This time an outside repository is used.  Runs on pull requests (PRs).

Create or modify a GitHub workflow, like below.  E.g. `.github/workflows/pr-open.yml`

```yaml
name: Pull Request

on:
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  builds:
    permissions:
      packages: write
    runs-on: ubuntu-24.04
    strategy:
      matrix:
        package: [backend, frontend]
        include:
          - package: backend
            triggers: ('backend/')
          - package: frontend
            triggers: ('frontend/')
    steps:
      - uses: actions/checkout@v4
      - name: Test Builds
        uses: bcgov/action-builder-ghcr@vX.Y.Z
        with:
          package: ${{ matrix.package }}
          tags: ${{ github.event.number }}
          tag_fallback: test
          repository: bcgov/nr-quickstart-typescript
          token: ${{ secrets.GITHUB_TOKEN }}
          triggers: ${{ matrix.triggers }}

```

# Outputs

New image digest (SHA).  This applies to build and retags.

```yaml
- id: digest
  uses: bcgov/action-builder-ghcr@vX.Y.Z
  ...

- name: Echo digest
  run: |
    echo "Digest: ${{ steps.digest.outputs.digest }}"
  ...
```

Has an image been built?  [true|false]

```yaml
- id: trigger
  uses: bcgov/action-builder-ghcr@vX.Y.Z
  ...

- name: Echo build trigger
  run: |
    echo "Trigger result: ${{ steps.trigger.outputs.triggered }}"
  ...
```

# Permissions

It is good practice to set explicit permissions for jobs and workflows.  These are applied to the GITHUB_TOKEN.  Please see the [GitHub documentation](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/controlling-permissions-for-github_token) for more information.

```yaml
permissions:
  packages: write
```

# Deprecations

- The `tag` input has been deprecated in favor of `tags`, a multiline string that can handle multiple values.
- The `digest_old` output has been deprecated due to non-use.
- The `digest_new` output has been renamed to `digest`.

<!-- # Acknowledgements

This Action is provided courtesy of the Forestry Suite of Applications, part of the Government of British Columbia. -->
