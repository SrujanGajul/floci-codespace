# Floci on GitHub Codespaces

Lightweight devcontainer that runs the [Floci](https://floci.io) AWS emulator
(native image) inside a free-tier GitHub Codespace and tunnels port `4566` back
to your local laptop, so the AWS CLI / SDKs on your machine talk to a real-AWS-API
running in the cloud.

- Codespace runtime: ~13 MiB idle (Floci native), sub-second startup
- Base image: `mcr.microsoft.com/devcontainers/base:ubuntu` (~600 MB, vs universal ~20 GB)
- Free-tier safe: fits `basicLinux32gb` (2-core / 4 GB / 32 GB)

---

## Architecture

```text
+---------------------+         +---------------------------------------+
|  Your local laptop  |         |        GitHub Codespace VM            |
|                     |  WSS    |                                       |
|  aws --endpoint-url |<-tunnel-+--+  Dev container (ubuntu-base)        |
|  http://:4566       |         |  +--+ docker-in-docker                |
|                     |         |     +--+ floci container :4566        |
+---------------------+         |        +--+ named volume floci-data   |
                                +---------------------------------------+
```

---

## One-time setup

### 0. Clone (skip if you already pushed it)

```bash
git clone git@github.com:SrujanGajul/floci-codespace.git
cd floci-codespace
```

### 1. gh CLI auth

Required scopes: `repo`, `codespace`.

```bash
gh auth login                       # interactive
gh auth status                      # verify
```

If you have **multiple accounts logged in** (e.g. work account via `GITHUB_TOKEN`
env var + personal account via interactive login), the env var wins. Override per
session:

```bash
unset GITHUB_TOKEN                  # use the interactively-logged-in account
gh auth status                      # confirm correct "Active account: true"
gh auth switch -u <username>        # if needed
```

Every command below assumes you've already done `unset GITHUB_TOKEN` if relevant.

---

## Lifecycle commands

| Action | Command |
|---|---|
| Create | `gh codespace create -R SrujanGajul/floci-codespace -m basicLinux32gb -b main --display-name floci` |
| List | `gh codespace list` |
| Open in VS Code (local) | `gh codespace code -c <name>` |
| Open in browser | `gh codespace view -c <name> --web` |
| SSH in | `gh codespace ssh -c <name>` |
| Run a one-off command | `gh codespace ssh -c <name> -- "docker ps"` |
| Forward port to local | `gh codespace ports forward 4566:4566 -c <name>` |
| List forwarded ports | `gh codespace ports -c <name>` |
| Make a port public | `gh codespace ports visibility 4566:public -c <name>` |
| Stop (pause) | `gh codespace stop -c <name>` |
| Resume | any `ssh` / `ports forward` / `code` auto-resumes; or `gh codespace ssh -c <name> -- true` |
| Rebuild container (apply devcontainer changes) | `gh codespace rebuild -c <name>` |
| Full rebuild (no cache) | `gh codespace rebuild --full -c <name>` |
| Change retention | `gh codespace edit -c <name> --retention-period 168h` |
| Delete | `gh codespace delete -c <name>` |

Replace `<name>` with the codespace ID from `gh codespace list` (e.g.
`floci-46wxp4w9wvpcp65`). Or just omit `-c` and pick interactively.

---

## Quick start: create → tunnel → use

```bash
unset GITHUB_TOKEN

# 1. Create (takes ~2-3 min the first time; ~20 sec on subsequent restarts)
gh codespace create \
  -R SrujanGajul/floci-codespace \
  -m basicLinux32gb \
  -b main \
  --display-name floci

# 2. Wait for state = Available
gh codespace list

# 3. Forward Floci's port to your local laptop (run in a dedicated terminal)
gh codespace ports forward 4566:4566 -c floci-<id>
# Leave this terminal running. Closing it tears down the tunnel.

# 4. In a NEW local terminal, set AWS env + smoke test
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test

aws s3 mb s3://test
echo hi | aws s3 cp - s3://test/hello.txt
aws s3 ls s3://test/
```

Persist the env vars across shells by adding the 4 `export` lines to
`~/.bashrc` / `~/.zshrc`.

---

## Pause / resume cycle (saves quota)

Codespaces auto-stop after **30 min idle** (account default, max 240 min). When
stopped:

- Container state preserved (volumes, files in `/workspaces/...`)
- Compute clock stops counting against monthly quota
- Tunnel drops (`websocket: close 1006`) — expected

To pause manually:

```bash
gh codespace stop -c floci-<id>
```

To resume:

```bash
# any of these wake the codespace:
gh codespace ports forward 4566:4566 -c floci-<id>
gh codespace ssh -c floci-<id>
gh codespace code -c floci-<id>
```

Floci container has `restart: unless-stopped`, so it auto-starts when the
codespace resumes. The `floci-data` named volume persists across stops.

---

## Data lifecycle

| Event | Floci data survives? |
|---|---|
| Tunnel disconnect | yes |
| Codespace stop / idle timeout | yes |
| Codespace resume | yes |
| `docker compose restart floci` inside codespace | yes |
| `gh codespace rebuild` | yes (named volume kept) |
| `gh codespace delete` | NO (volume gone with codespace) |

To export data before deleting:

```bash
gh codespace ssh -c floci-<id> -- "docker run --rm -v devcontainer_floci-data:/data -v /tmp:/backup alpine tar czf /backup/floci-data.tgz -C /data ."
gh codespace cp -e remote:/tmp/floci-data.tgz ./
```

---

## Common operations inside the codespace

SSH in first: `gh codespace ssh -c floci-<id>`

```bash
# status
docker compose -f .devcontainer/docker-compose.yml ps

# logs (follow)
docker compose -f .devcontainer/docker-compose.yml logs -f floci

# restart Floci
docker compose -f .devcontainer/docker-compose.yml restart floci

# stop / start
docker compose -f .devcontainer/docker-compose.yml stop
docker compose -f .devcontainer/docker-compose.yml up -d

# nuke + recreate
docker compose -f .devcontainer/docker-compose.yml down -v
docker compose -f .devcontainer/docker-compose.yml up -d
```

---

## Updating the devcontainer config

Edit `.devcontainer/devcontainer.json` or `.devcontainer/docker-compose.yml`,
commit, push:

```bash
git add .devcontainer/
git commit -m "chore: tweak devcontainer"
git push
```

Apply to a running codespace:

```bash
gh codespace rebuild -c floci-<id>           # uses cached layers
gh codespace rebuild --full -c floci-<id>    # no cache, rare
```

For brand-new codespaces, the new config is picked up automatically on `create`.

---

## Smoke test (full)

```bash
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test

# S3
aws s3 mb s3://e2e-test
echo "hello from local laptop" | aws s3 cp - s3://e2e-test/hello.txt
aws s3 ls s3://e2e-test/
aws s3 cp s3://e2e-test/hello.txt -

# SQS
aws sqs create-queue --queue-name orders
aws sqs send-message \
  --queue-url http://localhost:4566/000000000000/orders \
  --message-body '{"order":1}'
aws sqs receive-message --queue-url http://localhost:4566/000000000000/orders

# DynamoDB
aws dynamodb create-table \
  --table-name Users \
  --attribute-definitions AttributeName=id,AttributeType=S \
  --key-schema AttributeName=id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
aws dynamodb list-tables
aws dynamodb put-item --table-name Users --item '{"id":{"S":"u1"},"name":{"S":"Alice"}}'
aws dynamodb scan --table-name Users
```

All commands should return real AWS-shaped JSON / XML responses.

---

## Troubleshooting

### Tunnel exits with `websocket: close 1006 (abnormal closure): unexpected EOF`

Codespace went idle and was auto-stopped. Resume via any command above; reopen
the tunnel.

### `gh codespace ssh` fails: "SSH server is installed?"

The `sshd` feature is already in `.devcontainer/devcontainer.json`. If you ever
remove it and hit this error, add it back:

```json
"ghcr.io/devcontainers/features/sshd:1": { "version": "latest" }
```

Then `gh codespace rebuild`.

### Codespace boots into a "recovery container" (alpine, no Floci)

Logs show `Failed to create container` → `Creating recovery container`. The
prebuild phase glitched on the Codespaces backend. Delete and recreate:

```bash
gh codespace delete -c floci-<id> --force
gh codespace create -R SrujanGajul/floci-codespace -m basicLinux32gb -b main --display-name floci
```

Check logs to confirm a clean build:

```bash
gh codespace logs -c floci-<new-id> | grep -i 'recovery\|finished configuring\|Container floci'
```

### `Permission denied (publickey)` on `git push` after `gh repo create`

You created the repo under one GH account but your SSH key belongs to another.
Either switch accounts (`gh auth switch -u <user>`) before `gh repo create`, or
change the remote to HTTPS:

```bash
git remote set-url origin https://github.com/<user>/floci-codespace.git
```

`gh` will inject its token as the HTTPS credential helper.

### `gh codespace create` ignores `GITHUB_TOKEN`-bound account

The env var token shadows interactive auth. `unset GITHUB_TOKEN` for any
codespace command on your personal repo.

---

## Files in this repo

| File | Purpose |
|---|---|
| [.devcontainer/devcontainer.json](.devcontainer/devcontainer.json) | Devcontainer config: base image, features (DinD, aws-cli, sshd), env vars, port forward, postCreate |
| [.devcontainer/docker-compose.yml](.devcontainer/docker-compose.yml) | Floci service, port 4566, named volume |
| [README.md](README.md) | This document |
| [.gitignore](.gitignore) | Ignore local data dumps, .env |

---

## Notes

- The UFW caveat in upstream Floci docs (Lambda timing out on native Linux
  Docker hosts) does NOT apply here — Codespaces runs Floci inside
  docker-in-docker, not on a host Linux daemon with UFW.
- To swap to JVM image: change `floci/floci:latest` → `floci/floci:latest-jvm`
  in `.devcontainer/docker-compose.yml`, commit, `gh codespace rebuild`.
- Port 4566 is `private` (auto-forwarded URL only accessible to you when logged
  into GitHub). Make public via `gh codespace ports visibility 4566:public -c <name>`
  — but anything bound to `test/test` AWS creds is effectively unauthenticated,
  so don't expose to the internet unless you understand the trade-off.
