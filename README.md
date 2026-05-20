# rp-container-deploy-action

GitHub composite action to deploy arbitrary repositories to mittwald container hosting with close to zero configuration.

From a user perspective, this means you can take code that already works in your repository and get it running in container hosting without first designing Docker setup, build logic, or deployment scripts. You keep your attention on shipping software, while this action covers the path from repository to running container.

It is especially useful when speed and simplicity matter: beginners can deploy without deep hosting knowledge, and agentic or vibe coding workflows can move from idea to live service without getting blocked by infrastructure details.

## Usage

```yaml
- name: Deploy to Mittwald Container Hosting
  uses: mittwald/rp-container-deploy-action@master
  with:
    mittwald-api-token: ${{ secrets.MITTWALD_API_TOKEN }}
    mittwald-project-id: 'p-kpbj8e'
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `mittwald-api-token` | Mittwald API token for authentication | **yes** | – |
| `mittwald-project-id` | Mittwald project ID to deploy to (e.g. `p-kpbj8e`) | **yes** | – |
| `node-version` | Node.js version to use | no | `24` |
| `mittwald-cli-branch` | Branch of the mittwald CLI to clone | no | `feat/containerize-deploy` |
| `container-deploy-repo` | URL of the container-deploy repository | no | `https://github.com/mittwald/container-deploy` |
| `repo-subpath` | Subpath within the repository to deploy (e.g. `./services/api`), relative to the repository root | no | `.` |

## Environment file handling

This action automatically checks for a `.env` file in the repository root (`$GITHUB_WORKSPACE/.env`):

- If present, deployment is executed with `--env-file=$GITHUB_WORKSPACE/.env`.
- If not present, deployment runs without an environment file.

You can create this file in your calling workflow to construct the runtime environment for the deployed container.
On GitHub-hosted runners, this file only exists for the lifetime of the job workspace.

## Example Workflow

```yaml
name: Deploy to Mittwald Container Hosting

on:
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    name: Deploy

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Create .env for deployment
        env:
          APP_SECRET: ${{ secrets.APP_SECRET }}
        run: |
          {
            echo "APP_ENV=production"
            echo "APP_SECRET=$APP_SECRET"
          } > .env

      - name: Deploy to Mittwald Container Hosting
        uses: mittwald/rp-container-deploy-action@master
        with:
          mittwald-api-token: ${{ secrets.MITTWALD_API_TOKEN }}
          mittwald-project-id: 'p-kpbj8e'
```
