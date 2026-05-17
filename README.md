# Floci on GitHub Codespaces

Lightweight devcontainer that runs the [Floci](https://floci.io) AWS emulator
(native image) plus the [StackPort](https://github.com/DaviReisVieira/stackport)
web UI inside a free-tier GitHub Codespace. Tunnel ports `4566` (AWS API) and
`8080` (web UI) back to your local laptop.

- Codespace runtime: ~13 MiB idle (Floci) + ~150 MB (StackPort)
- Base image: `mcr.microsoft.com/devcontainers/base:ubuntu` (~600 MB, vs universal ~20 GB)
- Free-tier safe: fits `basicLinux32gb` (2-core / 4 GB / 32 GB)
- Web UI covers 35 AWS services with 8 dedicated rich browsers
  (S3, DynamoDB, Lambda, SQS, IAM, EC2, CloudWatch Logs, Secrets Manager)

---

## Architecture

```text
+---------------------+         +---------------------------------------+
|  Your local laptop  |         |        GitHub Codespace VM            |
|                     |  WSS    |                                       |
|  aws cli  :4566     |<-tunnel-+--+  Dev container (ubuntu-base)        |
|  browser  :8080     |<-tunnel-+  +--+ docker-in-docker                 |
|                     |         |     +--+ floci      :4566 (AWS API)   |
|                     |         |     +--+ stackport  :8080 (Web UI)    |
+---------------------+         |        +--+ floci-data named volume   |
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
| Forward floci and WebUI(StackPort) port to local | `gh codespace ports forward 4566:4566 8080:8080 -c <name>` |
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

# 3a. Forward BOTH ports to local laptop (one terminal)
gh codespace ports forward 4566:4566 8080:8080 -c floci-<id>
# Leave this terminal running. Closing it tears down the tunnel.

# 3b. Open the web UI in your browser
xdg-open http://localhost:8080      # Linux
# open http://localhost:8080        # macOS

# 4. In a NEW local terminal, use the AWS CLI directly
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test

aws s3 mb s3://test
echo hi | aws s3 cp - s3://test/hello.txt
aws s3 ls s3://test/
# Refresh StackPort browser tab to see the new bucket and object.
```

Persist the env vars across shells by adding the 4 `export` lines to
`~/.bashrc` / `~/.zshrc`.

## Web UI: StackPort

Browser-based AWS resource browser served from the codespace, fully open-source
([StackPort](https://github.com/DaviReisVieira/stackport), MIT license).

| URL | Notes |
|---|---|
| `http://localhost:8080` (via tunnel) | Recommended. Run `gh codespace ports forward 8080:8080 -c <name>` first. |
| `https://<codespace>-8080.app.github.dev` | Direct Codespaces auto-forwarded URL. Private to your GH session by default. Get exact URL via `gh codespace ports -c <name>`. |

Features available out of the box:

- **Dashboard** — service health + live resource counts across all 35 services
- **Dedicated browsers** — S3 (upload/download/folders), DynamoDB (query/scan), Lambda (invoke), SQS (send/receive), IAM, EC2, CloudWatch Logs, Secrets Manager
- **Generic resource table** — JSON detail + export for all other services
- **Tag management** — read/write tags across 21 resource types
- **Live updates** via WebSocket

`STACKPORT_ALLOW_WRITES=true` is set in `docker-compose.yml`, so the UI can
create/delete resources (e.g. drop S3 buckets, purge SQS queues). Flip to
`false` to enforce read-only mode.

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

### Robust tunnel pattern (wake-then-forward)

`gh codespace ports forward` auto-wakes the codespace, but if you run it the
instant the codespace is mid-`ShuttingDown` or just woken, the inner ports may
not be listening yet and you'll see:

```
error connecting to tunnel: ... ssh: rejected: connect failed (Connection refused)
```

Wake first, wait for compose, then tunnel:

```bash
unset GITHUB_TOKEN
gh codespace ssh -c floci-46wxp4w9wvpcp65 -- "true"   # wakes + waits
sleep 5                                                # let compose stabilize
gh codespace ports forward 4566:4566 -c floci-46wxp4w9wvpcp65 &
gh codespace ports forward 8080:8080 -c floci-46wxp4w9wvpcp65 &
```

Or simply retry the forward — the second attempt usually works because the
first one already woke the codespace.

For extra safety, wait for containers to actually be healthy:

```bash
unset GITHUB_TOKEN
CS=floci-46wxp4w9wvpcp65
gh codespace ssh -c $CS -- "true"
until gh codespace ssh -c $CS -- "docker ps --format '{{.Names}} {{.Status}}' | grep -q 'floci.*healthy'"; do
  echo "waiting for floci..."; sleep 3
done
gh codespace ports forward 4566:4566 -c $CS &
gh codespace ports forward 8080:8080 -c $CS &
```

### One-liner alias

Drop into `~/.bashrc` / `~/.zshrc`:

```bash
floci-up() {
  local CS=floci-46wxp4w9wvpcp65
  unset GITHUB_TOKEN
  gh codespace ssh -c "$CS" -- "true" && sleep 5
  gh codespace ports forward 4566:4566 -c "$CS" &
  gh codespace ports forward 8080:8080 -c "$CS" &
  echo "Tunnels backgrounded. http://localhost:8080 ready."
}

floci-down() {
  local CS=floci-46wxp4w9wvpcp65
  unset GITHUB_TOKEN
  pkill -f "gh codespace ports forward.*$CS"
  gh codespace stop -c "$CS"
}
```

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
| [.devcontainer/docker-compose.yml](.devcontainer/docker-compose.yml) | Floci service (`:4566`) + StackPort web UI (`:8080`), named volume |
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
