# Migrating from v1 to v2

v2 of this action is based on version 1.29.2 or newer of the mittwald CLI, which changed how deployments are named. Most workflows only need to change `@v1` to `@v2`, but **existing deployments are not updated in place by default**. Read the section on [service naming](#breaking-change-service-naming) before switching a production workflow.

## Summary of changes

- **Breaking:** the container service, its stack, the image name and the default domain are now named after the deployed directory instead of using fixed names.
- **Breaking:** the minimum supported mittwald CLI version is now 1.29.2 (default `mittwald-cli-version` is `^1.29.2`).
- New inputs: `service-name`, `uri-prefix`, `image-name`, `image-tag`, `env`, `env-file`, `wait-timeout`.
- New outputs: `service-name`, `url`.
- Unchanged: `.env` in the repository root is still picked up automatically if you don't set `env-file`.

## Breaking change: service naming

v1 deployed every repository into the same fixed names:

| | v1 | v2 default |
|---|---|---|
| Service name | `app-<project UUID>` | name of the deployed directory, e.g. `my-shop` |
| Stack | project's default stack | a stack of its own, named after the service |
| Image name | `app-image` | service name |
| Domain | `webapp.<project-short-id>.project.space` | `<service name>.<project-short-id>.project.space` |

The directory name is lowercased, and every run of characters other than letters and digits is replaced by a dash. With `actions/checkout`, this is the repository name, or the last segment of `repo-subpath` if you set one.

**What happens if you just switch to `@v2`:** the first v2 run creates a *new* service with a *new* URL next to the old one. The old v1 service keeps running unchanged until you remove it.

Note that v1 did not pin the CLI version (`^1.17.0`). v1 workflows that ran after the release of CLI 1.29 (2026-11-02) may therefore already have created a service under the new naming scheme.

You have two options.

### Option A: move to the new naming scheme (recommended)

1. Switch to `@v2` and run the workflow. A new service is created; the `url` output and the job log tell you where it is reachable.
2. Check that the new deployment works and point any custom domains or links to it.
3. Delete the old `app-<project UUID>` service in mStudio or with `mw container delete`.

This is the recommended option, because it lets you deploy multiple repositories or monorepo packages into one project without them overwriting each other.

### Option B: keep updating the existing v1 deployment

Set the inputs to the names v1 used. The CLI finds the existing service in the default stack and updates it there:

```yaml
- uses: mittwald/zerodeploy-action@v2
  with:
    mittwald-api-token: ${{ secrets.MITTWALD_API_TOKEN }}
    mittwald-project-id: 'p-XXXXXX'
    service-name: app-<project UUID>
    image-name: app-image
    uri-prefix: webapp
```

`service-name` needs the full project UUID (the long ID, not the `p-XXXXXX` short ID). You can find it with `mw project get p-XXXXXX`, or by looking up the name of the existing service in mStudio.

## Minimum CLI version

The `mittwald-cli-version` input now defaults to `^1.29.2`, and older versions are no longer supported. If you set this input explicitly to an older version, remove it or raise it to at least `1.29.2`.

## Environment variables

`.env` handling from v1 still works: if `env-file` is not set and `.env` exists in the repository root, it is used.

Instead of writing secrets to a `.env` file in a separate step, you can now pass them through the `env` input. A bare `KEY` takes the value from the environment of the calling step:

```yaml
# v1
- name: Create .env for deployment
  env:
    APP_SECRET: ${{ secrets.APP_SECRET }}
  run: |
    {
      echo "APP_ENV=production"
      echo "APP_SECRET=$APP_SECRET"
    } > .env

- uses: mittwald/zerodeploy-action@v1
  with:
    mittwald-api-token: ${{ secrets.MITTWALD_API_TOKEN }}
    mittwald-project-id: 'p-XXXXXX'
```

```yaml
# v2
- uses: mittwald/zerodeploy-action@v2
  env:
    APP_SECRET: ${{ secrets.APP_SECRET }}
  with:
    mittwald-api-token: ${{ secrets.MITTWALD_API_TOKEN }}
    mittwald-project-id: 'p-XXXXXX'
    env: |
      APP_ENV=production
      APP_SECRET
```

To use environment files other than `.env` in the repository root, for example in a monorepo, list them in `env-file` (one path per line, relative to the repository root).

## New outputs

The action now has a `url` output with the public URL of the deployed service, so you can link it to a GitHub environment:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - uses: actions/checkout@v4
      - id: deploy
        uses: mittwald/zerodeploy-action@v2
        with:
          mittwald-api-token: ${{ secrets.MITTWALD_API_TOKEN }}
          mittwald-project-id: 'p-XXXXXX'
```
