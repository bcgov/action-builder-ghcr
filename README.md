# action-builder-ghcr

This Action has moved to the centralized [bcgov/actions](https://github.com/bcgov/actions) repository.

Please update your workflows to use:
```yaml
- name: Build & Push Image
  uses: bcgov/actions/builder-ghcr@v0.2.0
  with:
    package: my-app
    tags: |
      latest
```

This repository is kept as a wrapper to point to the new location and reduce disruption for existing pipelines (some inputs/outputs may differ from earlier releases).
