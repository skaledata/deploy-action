# SkaleData Deploy Action

Build, push, and deploy custom images to your SkaleData cluster. Works with GCP, AWS, and Azure — no cloud credentials needed.

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

      - uses: skaledata/deploy-action@v1
        with:
          api-key: ${{ secrets.SKALEDATA_API_KEY }}
          cluster-id: your-cluster-public-id
          app-type: airflow
          image-tag: ${{ github.sha }}
```

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
| `skip-build` | No | `false` | Skip build/push, only trigger deploy |

## Outputs

| Output | Description |
|--------|-------------|
| `image` | Full image URI that was deployed |
| `status` | Deploy status (`deployed` or `upgrading`) |

## How it works

1. Calls the SkaleData API to get short-lived registry credentials
2. Logs into the cluster's container registry (GCP Artifact Registry, AWS ECR, or Azure ACR)
3. Builds and pushes your Docker image
4. Triggers a deploy via the SkaleData API (rolling restart or Terraform upgrade)

No `gcloud`, `aws`, or `az` CLI needed. The API key is the only secret.

## Example Dockerfile

```dockerfile
FROM apache/airflow:3.1.6
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

## License

MIT
