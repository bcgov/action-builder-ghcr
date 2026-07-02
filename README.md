# action-builder-ghcr

This Action has moved to the centralized [bcgov/actions](https://github.com/bcgov/actions) repository.

Please update your workflows to use:
```yaml
- name: Build & Push Image
  uses: bcgov/actions/builder-ghcr@vX.Y.Z
  with:
    package: my-app
    tags: |
      latest
```

This repository is kept as a wrapper to point to the new location and prevent breaking existing pipelines.
