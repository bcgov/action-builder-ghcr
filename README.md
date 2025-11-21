<!-- Badges -->
[![Issues](https://img.shields.io/github/issues/bcgov/action-builder-ghcr)](/../../issues)
[![Pull Requests](https://img.shields.io/github/issues-pr/bcgov/action-builder-ghcr)](/../../pulls)
[![MIT License](https://img.shields.io/github/license/bcgov/action-builder-ghcr.svg)](/LICENSE)
[![Lifecycle](https://img.shields.io/badge/Lifecycle-Experimental-339999)](https://github.com/bcgov/repomountie/blob/master/doc/lifecycle-badges.md)

# Conditional Container Builder with Fallback, Attestations and SBOMs (Software Bill of Materials)

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
    # Note: All tags are normalized to lowercase and stripped of spaces before use.
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

    # SBOM generation is enabled by default as a security best practice
    # String value, not boolean
    sbom: 'true'

    # Specify token (GH or PAT), instead of inheriting one from the calling workflow
    token: ${{ secrets.GITHUB_TOKEN }}

    # Multiline input for secrets to mount.
    # https://docs.docker.com/build/ci/github-actions/secrets/#secret-mounts
    secrets: |
        MY_SECRET=${{ secrets.MY_SECRET }}
        ANOTHER_SECRET=${{ secrets.ANOTHER_SECRET }}

    # Enable automatic tag and label generation using docker/metadata-action
    # String value, not boolean. Defaults to 'false'
    # When enabled, generates tags based on branch names, semver tags, SHAs, and PR numbers
    metadata_tags: 'true'

    # Flavor configuration for metadata-action (optional)
    # Only used when metadata_tags is enabled
    metadata_flavor: |
        latest=true

    # Custom tag rules for metadata-action (optional)
    # Only used when metadata_tags is enabled
    # Tag rules are required when metadata_tags is enabled; no tags will be generated unless explicit rules are provided
    metadata_tag_rules: |
        type=ref,event=branch
        type=ref,event=pr
        type=semver,pattern={{version}}
        type=semver,pattern={{major}}.{{minor}}
        type=semver,pattern={{major}}
        type=sha

    ### Deprecated

    # Single-value tag input has been deprecated and will be removed in a future release
    # Please use inputs.tags, which can handle multiple values
    tag: do not use!

```

# Example, Single Build

Build a single subfolder with a Dockerfile in it.  Runs on pull requests (PRs).

```yaml
builds:
  runs-on: ubuntu-24.04
  steps:
    - name: Builds
      uses: bcgov/action-builder-ghcr@vX.Y.Z
      with:
        package: frontend
        tag_fallback: test
        triggers: ('frontend/')
```

# Example, Single Build with build_context, build_file and multiple tags

Same as previous, but specifying build folder and Dockerfile.

```yaml
builds:
  runs-on: ubuntu-24.04
  steps:
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

```yaml
builds:
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

# Example, Metadata Tags for Automatic Tagging

Use docker/metadata-action to automatically generate tags and labels based on branch names, semantic versions, PRs, and commit SHAs.

```yaml
builds:
  runs-on: ubuntu-24.04
  steps:
    - name: Builds with Metadata Tags
      uses: bcgov/action-builder-ghcr@vX.Y.Z
      with:
        package: frontend
        tag_fallback: test
        triggers: ('frontend/')
        # Enable metadata-action integration
        metadata_tags: 'true'
        # Configure automatic 'latest' tag
        metadata_flavor: |
          latest=true
        # Define tagging rules
        metadata_tag_rules: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=semver,pattern={{major}}
          type=sha
```

This will generate tags like:
- For branch pushes: `main`, `develop`, etc.
- For PRs: `pr-123`
- For semver tags: `v1.2.3`, `1.2`, `1`, `latest`
- For all commits: `sha-abc1234`

# Security Features

This action provides two key security features: Container Attestations and Software Bill of Materials (SBOM) generation. Additionally, it leverages the `docker/metadata-action` for best-practice container image tagging and labeling.

## Container Metadata and Labeling

This action uses the [`docker/metadata-action`](https://github.com/docker/metadata-action) to automatically generate OCI-compliant labels and annotations for container images. This ensures that images are tagged and labeled following [Open Container Initiative (OCI) specifications](https://specs.opencontainers.org/image-spec/annotations/) and Docker best practices.

The following [OCI Image Format Specification](https://github.com/opencontainers/image-spec/blob/main/annotations.md) labels are automatically added to all container images:

- **org.opencontainers.image.created** - Image creation timestamp (ISO 8601 format)
- **org.opencontainers.image.url** - URL to the source repository (e.g., https://github.com/bcgov/repo-name)
- **org.opencontainers.image.source** - URL to the source repository
- **org.opencontainers.image.version** - Version/tag of the image (e.g., main, pr-123)
- **org.opencontainers.image.revision** - Git commit SHA that triggered the build
- **org.opencontainers.image.title** - Human-readable title (repository name)
- **org.opencontainers.image.description** - Description from the repository
- **org.opencontainers.image.licenses** - License information from the repository's LICENSE file

These labels provide standardized metadata for source provenance, attestation, and help ensure baseline security practices for container deployments.

### Example OCI Labels

Here's an example of the OCI labels that would be applied to a container image built with this action:

```json
{
  "org.opencontainers.image.created": "2025-05-29T19:09:03.374Z",
  "org.opencontainers.image.url": "https://github.com/bcgov/nr-peach",
  "org.opencontainers.image.source": "https://github.com/bcgov/nr-peach",
  "org.opencontainers.image.version": "main",
  "org.opencontainers.image.revision": "fadf03ce2db919752ada03af3f8fb4895fe96fcf",
  "org.opencontainers.image.title": "nr-peach",
  "org.opencontainers.image.description": "NR Permitting Exchange, Aggregation and Collection Hub",
  "org.opencontainers.image.licenses": "Apache-2.0"
}
```

You can inspect these labels on any built image using:
```bash
docker inspect ghcr.io/org/repo/package:tag | jq '.[0].Config.Labels'
```

## Container Attestations

[Container attestations](https://docs.github.com/en/actions/security-guides/security-hardening-with-openid-connect#about-oidc-and-container-signing) use GitHub's OIDC token to provide cryptographic proof of:
- Where the container was built (GitHub Actions)
- When it was built (timestamp)
- What repository and workflow built it
- What inputs and environment were used

Attestations require the following permissions:
```yaml
permissions:
  packages: write      # Required for pushing images
  id-token: write      # Required for OIDC token generation
  attestations: write  # Required for creating attestations
```

If these permissions are not granted, the action will still build and push images but skip the attestation step.

## Software Bill of Materials (SBOM)

This action automatically generates SBOMs for all container builds using [Syft](https://github.com/anchore/syft). SBOMs provide a detailed inventory that includes:
- All installed packages and their versions
- Dependencies and their relationships
- License information
- Known vulnerabilities

Two SBOM formats are generated and uploaded as workflow artifacts:
- CycloneDX JSON
- SPDX JSON

# Outputs

| Output     | Description                                 |
|------------|---------------------------------------------|
| `digest`   | Digest of the built or retagged image       |
| `triggered`| Whether a build was triggered (`true/false`)|
| `labels`   | OCI labels generated by metadata-action (when enabled) |

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

# Image Naming Convention

- Multi package repos: `ghcr.io/org/repo/package:tag`
- Single package repos: `ghcr.io/org/repo:tag`

Single package naming is only triggered when package=repository.

# Deprecations

> ⚠️ **Deprecated:** The `tag` input has been deprecated in favor of `tags`, a multiline string that can handle multiple values. The `tag` input will be removed in a future release.

- The `digest_old` output has been deprecated due to non-use.
- The `digest_new` output has been renamed to `digest`.

<!-- # Acknowledgements

This Action is provided courtesy of the Forestry Suite of Applications, part of the Government of British Columbia. -->
