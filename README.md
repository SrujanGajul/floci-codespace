# Floci on GitHub Codespaces

Lightweight devcontainer that auto-starts the [Floci](https://floci.io) AWS
emulator (native image) on port `4566` inside a GitHub Codespace.

Designed for the 2-core / 4 GB free-tier Codespace. Floci native runs at ~13 MiB
idle with sub-second startup, so there is plenty of headroom for your own work.

## Quick start

1. Push this repo to GitHub (already done if you are reading this on github.com).
2. Click **Code → Codespaces → Create codespace on main**.
3. Wait for the dev container to build. The `postCreateCommand` automatically
   runs `docker compose up -d`, so Floci is listening on `localhost:4566` by the
   time the terminal opens.
4. AWS env vars (`AWS_ENDPOINT_URL`, `AWS_DEFAULT_REGION`, `AWS_ACCESS_KEY_ID`,
   `AWS_SECRET_ACCESS_KEY`) are pre-set in the container, so the `aws` CLI just
   works.

## Smoke test

```bash
# S3
aws s3 mb s3://test-bucket
echo "hi" | aws s3 cp - s3://test-bucket/hello.txt
aws s3 ls s3://test-bucket

# SQS
aws sqs create-queue --queue-name orders
aws sqs list-queues

# DynamoDB
aws dynamodb create-table \
  --table-name Users \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
aws dynamodb list-tables
```

## Managing Floci

```bash
# status
docker compose -f .devcontainer/docker-compose.yml ps

# logs
docker compose -f .devcontainer/docker-compose.yml logs -f floci

# restart
docker compose -f .devcontainer/docker-compose.yml restart floci

# stop / start
docker compose -f .devcontainer/docker-compose.yml stop
docker compose -f .devcontainer/docker-compose.yml up -d
```

## What's inside

| Piece | Purpose |
| --- | --- |
| `mcr.microsoft.com/devcontainers/base:ubuntu` | Slim base image (~600 MB vs universal ~20 GB) |
| `docker-in-docker` feature | Inner Docker daemon so Floci can run |
| `aws-cli` feature | AWS CLI v2 inside the dev container |
| `.devcontainer/docker-compose.yml` | Pulls `floci/floci:latest` (native), maps `4566`, persists `/app/data` to a named volume |
| `containerEnv` | Pre-set AWS endpoint + dummy creds |
| `forwardPorts: [4566]` | Codespaces auto-forwards Floci's port to your browser |

## Data persistence

The named volume `floci-data` survives Codespace **rebuilds**. It is lost only
when you delete the Codespace itself. Stop and restart any time.

## Notes

- The UFW caveat in the Floci docs does **not** apply here — Codespaces runs
  Floci inside docker-in-docker, not on a host Linux daemon.
- If you ever need the JVM variant, swap `floci/floci:latest` for
  `floci/floci:latest-jvm` in `.devcontainer/docker-compose.yml`.
