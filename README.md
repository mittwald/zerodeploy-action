# zerodeploy-action

GitHub composite action to deploy arbitrary repositories to mittwald container hosting with close to zero configuration.

From a user perspective, this means you can take code that already works in your repository and get it running in container hosting without first designing Docker setup, build logic, or deployment scripts. You keep your attention on shipping software, while this action covers the path from repository to running container.

It is especially useful when speed and simplicity matter: beginners can deploy without deep hosting knowledge, and agentic or vibe coding workflows can move from idea to live service without getting blocked by infrastructure details.

> [!NOTE]
> Upgrading from v1? See the [v2 migration guide](MIGRATION-v2.md).

## Usage

```yaml
- name: Deploy to Mittwald Container Hosting
  uses: mittwald/zerodeploy-action@v2
  with:
    mittwald-api-token: ${{ secrets.MITTWALD_API_TOKEN }}
    mittwald-project-id: 'p-XXXXXX'
```

The action builds the application (using its `Dockerfile` if there is one, and [Railpack](https://railpack.com) otherwise), pushes the image to a private registry in your project, deploys it as a container service and makes it available at `https://<uri-prefix>.<project-short-id>.project.space`.

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `mittwald-api-token` | Mittwald API token for authentication | **yes** | – |
| `mittwald-project-id` | Mittwald project ID to deploy to (e.g. `p-XXXXXX`) | **yes** | – |
| `repo-subpath` | Subpath within the repository to deploy (e.g. `./services/api`), relative to the repository root | no | `.` |
| `service-name` | Name of the container service (and of the stack it runs in). Deployments with different service names run independently of each other in the same project; deploying again with the same name updates the existing service. | no | name of the deployed directory (see [below](#service-naming)) |
| `uri-prefix` | Prefix for the generated default domain `<uri-prefix>.<project-short-id>.project.space` | no | service name |
| `image-name` | Name of the container image to build and push | no | service name |
| `image-tag` | Tag of the container image to build and push, e.g. `${{ github.sha }}` | no | `latest` |
| `env` | Environment variables for the container, one per line, as `KEY=VALUE` or `KEY` (see [Environment variables](#environment-variables)) | no | – |
| `env-file` | Paths to environment files, one per line, relative to the repository root | no | `.env`, if it exists |
| `wait-timeout` | How long to wait for the registry, service and domain to become ready (e.g. `600s`, `15m`) | no | `600s` |
| `build-only` | Only build and push the image, without deploying a service or creating a domain | no | `false` |
| `node-version` | Node.js version to use. Typically only used for development and testing purposes. | no | `24` |
| `mittwald-cli-version` | Version selector for mittwald CLI. Typically only used for development and testing purposes. Versions below 1.29.2 are not supported. | no | `^1.29.2` |

## Outputs

| Output | Description |
|--------|-------------|
| `service-name` | Name of the deployed container service |
| `url` | Public URL of the deployed service, e.g. `https://my-app.p-xxxxxx.project.space`. Empty if `build-only` is set. |

## Service naming

If you don't set `service-name`, it is derived from the name of the deployed directory: the name is lowercased and every run of characters other than letters and digits is replaced by a dash. With `actions/checkout`, the repository is checked out into a directory named after the repository, so:

- deploying the repository root of `acme/My_Shop` creates the service `my-shop`
- deploying `repo-subpath: ./services/api` creates the service `api`

The image name and domain prefix default to the service name, so the example above ends up at `https://my-shop.<project-short-id>.project.space`.

Since every service runs in a stack of its own, you can deploy several independent services into the same project, for example one per monorepo package or one per pull request, just by giving them different service names.

## Environment variables

There are two ways to pass environment variables to the container, and you can combine them.

**`env` input:** one variable per line. `KEY=VALUE` sets the value directly. A bare `KEY` takes the value from the environment of the calling step, which is the recommended way to pass secrets:

```yaml
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

Empty lines and lines starting with `#` are ignored.

**Environment files:** files with one `KEY=VALUE` per line. List them in the `env-file` input (one path per line, relative to the repository root). A file listed there that does not exist fails the deployment. If you don't set `env-file`, the action uses `.env` in the repository root, if it exists.

If a variable is defined in both places, the value from the environment file wins.

On GitHub-hosted runners, files you create in the workspace only exist for the lifetime of the job.

## Examples

### Deploy on every push to the main branch

```yaml
name: Deploy to Mittwald Container Hosting

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    name: Deploy
    environment:
      name: production
      url: ${{ steps.deploy.outputs.url }}

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Deploy to Mittwald Container Hosting
        id: deploy
        uses: mittwald/zerodeploy-action@v2
        env:
          APP_SECRET: ${{ secrets.APP_SECRET }}
        with:
          mittwald-api-token: ${{ secrets.MITTWALD_API_TOKEN }}
          mittwald-project-id: 'p-XXXXXX'
          image-tag: ${{ github.sha }}
          env: |
            APP_ENV=production
            APP_SECRET
```

### Preview deployment per pull request

```yaml
name: Preview deployment

on:
  pull_request:

jobs:
  preview:
    runs-on: ubuntu-latest
    environment:
      name: preview-pr-${{ github.event.number }}
      url: ${{ steps.deploy.outputs.url }}

    steps:
      - uses: actions/checkout@v4

      - name: Deploy preview
        id: deploy
        uses: mittwald/zerodeploy-action@v2
        with:
          mittwald-api-token: ${{ secrets.MITTWALD_API_TOKEN }}
          mittwald-project-id: 'p-XXXXXX'
          service-name: pr-${{ github.event.number }}
          image-tag: ${{ github.sha }}
```

### Monorepo with multiple services

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    strategy:
      # all services share the project registry, which is created on the
      # first deployment; deploy one at a time to avoid racing on its setup
      max-parallel: 1
      matrix:
        service: [api, frontend]

    steps:
      - uses: actions/checkout@v4

      - uses: mittwald/zerodeploy-action@v2
        with:
          mittwald-api-token: ${{ secrets.MITTWALD_API_TOKEN }}
          mittwald-project-id: 'p-XXXXXX'
          repo-subpath: ./services/${{ matrix.service }}
          env-file: services/${{ matrix.service }}/.env.production
```
