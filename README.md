# Builder GHCR (Moved)

> [!IMPORTANT]
> **This action has moved to [`bcgov/actions/builder`](https://github.com/bcgov/actions/tree/main/builder).** This repository is retiring and will be archived. Update workflows to the new path, pinned to a [release](https://github.com/bcgov/actions/releases) (not `@main`):
>
> ```yaml
> - uses: bcgov/actions/builder@vX.Y.Z
>   with:
>     package: backend
>     tag_fallback: latest
>     triggers: |
>       backend/
> ```
>
> | Old input | New input |
> | --- | --- |
> | `github_token` | `token` |
> | `diff_branch` | `ref` |
> | `metadata_tags`, `metadata_flavor`, `metadata_tag_rules` | Removed. OCI labels are always applied; custom tag rules are gone. |
>
> Other inputs keep their names. See [Migrating from action-builder-ghcr](https://github.com/bcgov/actions/tree/main/builder#migrating-from-action-builder-ghcr).
