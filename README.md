# SkaleData Deploy Action

Build/push custom images **and** sync DAGs to your SkaleData cluster. Works with GCP, AWS, and Azure — no cloud credentials needed.

Starting with **v2**, the action does two things on each run:

1. **Image rebuild + deploy** (slow, ~2-5 min) — only when files in `image-rebuild-paths` actually changed in the push.
2. **DAG upload** (fast, ~5s) — always, regardless of whether the image rebuilt. The sync sidecar pulls DAGs into pods within ~30s.

This means a one-line DAG edit doesn't trigger an image rebuild, but DAGs are always kept in lockstep with `main`.

## Usage

```yaml
name: Deploy Airflow
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # needed for diff-based change detection

      - uses: skaledata/deploy-action@v2
        with:
          api-key: ${{ secrets.SKALEDATA_API_KEY }}
          cluster-id: your-cluster-public-id
          app-type: airflow
```

> **Note on `fetch-depth`:** the default `actions/checkout` shallow clone (depth=1) doesn't include the previous commit, so change detection can't compute a diff and falls back to "rebuild everything". Use `fetch-depth: 0` (full history) or `fetch-depth: 2` (just enough for `HEAD^`) if you want change detection to work on push events.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `api-key` | Yes | — | SkaleData API key (`sdk_...`) |
| `cluster-id` | Yes | — | Cluster public ID from the SkaleData console |
| `app-type` | Yes | `airflow` | Application type (`airflow`, `airbyte`, `docs`, `slackbot`, `superset`, `datahub`) |
| `image-tag` | No | `${{ github.sha }}` | Docker image tag |
| `dockerfile` | No | `./Dockerfile` | Path to Dockerfile |
| `context` | No | `.` | Docker build context directory |
| `build-args` | No | — | Docker build arguments (one per line, `KEY=VALUE`) |
| `api-url` | No | `https://api.skaledata.com` | SkaleData API URL |
| `skip-build` | No | `false` | Skip build/push/deploy-image, even when image-rebuild paths match. DAGs still upload. |
| `image-rebuild-paths` | No | *see below* | Newline-separated glob patterns. If any changed file matches, the image is rebuilt. |
| `dags-path` | No | `dags` | Path to DAGs folder, tarred and uploaded on every run. |
| `skip-dag-upload` | No | `false` | Skip the DAG sync upload (rare — only if DAGs are managed out-of-band). |

### Default `image-rebuild-paths`

```text
Dockerfile
requirements.txt
requirements/**
pyproject.toml
setup.py
setup.cfg
packages/**
plugins/**
```

Override per repo as needed (e.g. add `Pipfile.lock`, drop `plugins/**` if you don't ship custom plugins).

## Outputs

| Output | Description |
|--------|-------------|
| `image` | Full image URI that was deployed (empty when image was not rebuilt) |
| `status` | `deployed`, `upgrading`, or empty when rebuild was skipped |
| `image-rebuilt` | `true` if an image build + deploy ran this invocation |
| `dag-files-uploaded` | Number of DAG files uploaded to blob storage |
| `dag-sync-url` | Blob storage URL DAGs synced to |

## How it works

1. **Detect changes**: diffs the push against the previous commit (or PR base) and matches changed files against `image-rebuild-paths`. Falls back to "rebuild" when no diff base is available (e.g. `workflow_dispatch`, first push to branch).
2. **Image path** (only if `image-rebuild=true` and `skip-build != true`):
    - Mint short-lived registry credentials from the SkaleData API
    - Log in to the cluster's container registry (GCP Artifact Registry / AWS ECR / Azure ACR)
    - Build and push the image (`:<sha>` + `:latest`)
    - Trigger rolling restart or first-time Terraform upgrade via `POST /clusters/{id}/deploy-image`
3. **DAG path** (always, unless `skip-dag-upload=true`):
    - Tar the `dags-path` folder
    - `POST /clusters/{id}/upload-dags` with the tar.gz + `app_type`
    - The cluster's sync sidecar picks up the DAGs within ~30s

No `gcloud`, `aws`, or `az` CLI needed. The API key is the only secret.

## Example Dockerfile

Plugins are baked into the image; DAGs are *not* (they come from blob sync):

```dockerfile
FROM apache/airflow:3.2.2-python3.12

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Plugins ship in the image. DAGs sync from blob storage via the sidecar.
COPY --chown=airflow:root plugins/ /opt/airflow/plugins/
```

## Migrating from v1

v1 always built and pushed an image; there was no DAG sync. To upgrade:

1. Bump `uses: skaledata/deploy-action@v2`
2. Add `fetch-depth: 0` to your `actions/checkout` step
3. Remove any `COPY dags/` line from your Dockerfile (DAGs no longer belong in the image; the sidecar overwrites that path on pod startup)
4. (Optional) Customize `image-rebuild-paths` if your repo has non-standard file layouts

If you want v1 behavior (rebuild on every push), set `image-rebuild-paths: '*'` and `skip-dag-upload: 'true'`.

## License

MIT
