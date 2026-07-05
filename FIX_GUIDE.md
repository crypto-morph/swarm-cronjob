# Swarm-Cronjob Fix Guide (crypto-morph fork)

## Problem

After upgrading Docker Engine (especially v29+), `voinetwork_scheduler` crash-loops every ~5 seconds with:

```
Cannot initialize swarm-cronjob error="Error response from daemon: client version 1.28 is too old. Minimum supported API version is 1.40, please upgrade your client to a newer version"
```

This is the same class of bug as [crazy-max/swarm-cronjob#307](https://github.com/crazy-max/swarm-cronjob/issues/307): the image embeds a Docker API client pinned to an old version that newer Docker daemons reject.

The VoiNetwork `:edge` image on GHCR still pins API `1.28`. Docker 29 requires API `1.40` minimum.

## Recommended fix: use the public image

A fixed image is published from this fork:

```bash
docker pull ghcr.io/crypto-morph/swarm-cronjob:edge
docker service update --image ghcr.io/crypto-morph/swarm-cronjob:edge voinetwork_scheduler
```

Verify:

```bash
docker service ls | grep scheduler          # expect 1/1
docker service logs voinetwork_scheduler --tail 5
```

Expected log lines:

```
INF Starting swarm-cronjob <commit>
INF Add cronjob with schedule ... service=voinetwork_shepherd
```

## Update compose for future stack deploys

In `docker/compose.yml` (or profile-specific yml), set:

```yaml
scheduler:
  image: ghcr.io/crypto-morph/swarm-cronjob:edge
```

Then redeploy if needed:

```bash
docker stack deploy -c docker/compose.yml voinetwork
```

## Does algod still auto-update?

Yes. The update chain is unchanged:

1. **scheduler** (swarm-cronjob) — runs continuously, triggers shepherd on cron
2. **shepherd** — runs once per trigger, pulls `:latest` images and updates swarm services
3. **algod** — updated by shepherd (`ghcr.io/voinetwork/voi-node-participation-mainnet:latest`)

Shepherd is intentionally `replicas: 0`; it only runs when the scheduler fires its cron job.

## Build from source (optional)

Repository: https://github.com/crypto-morph/swarm-cronjob

```bash
git clone https://github.com/crypto-morph/swarm-cronjob.git
cd swarm-cronjob
docker build -t ghcr.io/crypto-morph/swarm-cronjob:edge .
```

CI also builds and pushes `:edge` on every push to `master`.

## What was fixed in code

Two changes in `internal/docker/client.go`:

1. **API version negotiation** — replace `client.WithVersion("1.28")` with `client.WithAPIVersionNegotiation()`
2. **Docker CLI init for Swarm** — use minimal `command.NewDockerCli()` options for container environments

See branch `modernize-docker27-go123` history for details.

## Quick diagnostic

```bash
docker service ps voinetwork_scheduler --no-trunc | grep -i failed
docker service logs voinetwork_scheduler --tail 20
docker version   # check Server API version
```

If you see repeated `Failed` tasks and the API version error above, switch to `ghcr.io/crypto-morph/swarm-cronjob:edge`.
