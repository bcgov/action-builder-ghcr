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

    # Tag name (<package>:<tag>)
    tag: ${{ github.event.number }}


    ### Typical / recommended

    # Fallback tag, used if no build was generated
    # Optional, defaults to nothing, which forces a build
    # Non-matching or malformed tags are rejected, which also forced a build
    tag_fallback: test

    # Bash array to diff for build triggering
    # Optional, defaults to nothing, which forces a build
    triggers: ('frontend/' 'backend/' 'database/')

    # Sets the build context/directory, which contains the build files
    # Optional, defaults to package name
    build_context: ./frontend

    # Sets the Dockerfile with path
    # Optional, defaults to {package}/Dockerfile or {build_context}/Dockerfile
    build_file: ./frontend/Dockerfile

    # Number of packages to keep if cleaning up previous builds
    # Optional, skips if not provided
    keep_versions: 50


    ### Usually a bad idea / not recommended

    # Sets a list of [build-time variables](https://docs.docker.com/engine/reference/commandline/buildx_build/#build-arg)
    # Optional, defaults to sample content
    build_args: |
      ENV=build

    # Overrides the default branch to diff against
    # Defaults to the default branch, usually `main`
    diff_branch: ${{ github.event.repository.default_branch }}

    # Regex for tags to skip when cleaning up packages; defaults to test and prod
    # Only used when keep_versions is provided
    keep_regex: "^(prod|test)$"

    # Repository to clone and process
    # Useful for consuming other repos, like in testing
    # Defaults to the current one
    repository: ${{ github.repository }}

    # Specify token (GH or PAT), instead of inheriting one from the calling workflow
    token: ${{ secrets.GITHUB_TOKEN }}

```

# Example, Single Build

Build a single subfolder with a Dockerfile in it.  Deletes old packages, keeping the last 50.  Runs on pull requests (PRs).

Create or modify a GitHub workflow, like below.  E.g. `./github/workflows/pr-open.yml`

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
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v3
      - name: Builds
        uses: bcgov/action-builder-ghcr@vX.Y.Z
        with:
          package: frontend
          keep_versions: 50
          tag: ${{ github.event.number }}
          tag_fallback: test
          token: ${{ secrets.GITHUB_TOKEN }}
          triggers: ('frontend/')
```

# Example, Single Build with build_context and build_file

Same as previous, but specifying build folder and Dockerfile.

Create or modify a GitHub workflow, like below.  E.g. `./github/workflows/pr-open.yml`

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
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v3
      - name: Builds
        uses: bcgov/action-builder-ghcr@vX.Y.Z
        with:
          package: frontend
          build_context: ./
          build_file: subdir/Dockerfile
          keep_versions: 50
          tag: ${{ github.event.number }}
          tag_fallback: test
          token: ${{ secrets.GITHUB_TOKEN }}
          triggers: ('frontend/')
```

# Example, Matrix Build

Build from multiple subfolders with Dockerfile in them.  This time an outside repository is used.  Runs on pull requests (PRs).

Create or modify a GitHub workflow, like below.  E.g. `./github/workflows/pr-open.yml`

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
    runs-on: ubuntu-22.04
    strategy:
      matrix:
        package: [backend, frontend]
        include:
          - package: backend
            triggers: ('backend/')
          - package: frontend
            triggers: ('frontend/')
    steps:
      - uses: actions/checkout@v3
      - name: Test Builds
        uses: bcgov/action-builder-ghcr@vX.Y.Z
        with:
          package: ${{ matrix.package }}
          tag: ${{ github.event.number }}
          tag_fallback: test
          repository: bcgov/nr-quickstart-typescript
          token: ${{ secrets.GITHUB_TOKEN }}
          triggers: ${{ matrix.triggers }}

```

# Outputs

Returns digests for the new and previous images, if available.  This applies to build and retags.

```yaml
- id: meaningful_id_name
  uses: bcgov/action-builder-ghcr@vX.Y.Z
  ...

- name: Echo digest
  run: |
    echo "Digest, new: ${{ steps.meaningful_id_name.outputs.digest }}"
    echo "Digest, old: ${{ steps.meaningful_id_name.outputs.digest_old }}"
  ...
```

Has an image been built?  [true|false]

```yaml
- id: meaningful_id_name
  uses: bcgov/action-builder-ghcr@vX.Y.Z
  ...

- name: Echo build trigger
  run: |
    echo "Trigger result: ${{ steps.meaningful_id_name.outputs.triggered }}"
  ...
```

# Permissions

Workflows kicked off by Dependabot or a fork run with reduced permissions.  That can be addressed by setting explict permissions for the GITHUB_TOKEN.  If this is not required, then remove the lines below from these examples.

```yaml
permissions:
  packages: write
```

<!-- # Acknowledgements

This Action is provided courtesty of the Forestry Suite of Applications, part of the Government of British Columbia. -->
