# Dashy + Traefik + Let's Encrypt on Docker Compose

[![Deployment Verification](https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml/badge.svg?branch=main)](https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml)

This repository deploys **Dashy 4** (a self-hosted dashboard for all your services) behind **Traefik** with automatic **Let's Encrypt TLS**. One `docker compose up` away from your own start page at `https://your-domain`.

📙 Full narrative installation guide on the blog: [heyvaldemar.com/install-dashy-using-docker-compose/](https://www.heyvaldemar.com/install-dashy-using-docker-compose/).

## Getting started

```bash
# 1. Clone
git clone https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose
cd dashy-traefik-letsencrypt-docker-compose

# 2. Create the two Docker networks the stack expects
docker network create traefik-network
docker network create dashy-network

# 3. Copy the environment template and fill in required values
cp .env.example .env
$EDITOR .env
# ^ Required: TRAEFIK_ACME_EMAIL, TRAEFIK_HOSTNAME, TRAEFIK_BASIC_AUTH,
#   DASHY_HOSTNAME. See .env.example for generation commands.

# 4. Put your dashboard content into config.yml (the shipped one is a starter)
$EDITOR config.yml

# 5. Deploy
docker compose -f dashy-traefik-letsencrypt-docker-compose.yml -p dashy up -d
```

Within a minute or two, `https://${DASHY_HOSTNAME}` serves your dashboard and `https://${TRAEFIK_HOSTNAME}` serves the basic-auth protected Traefik dashboard, both with fresh Let's Encrypt certificates.

### What success looks like

```bash
docker compose -f dashy-traefik-letsencrypt-docker-compose.yml -p dashy ps
# Expected: dashy and traefik both show "(healthy)"

curl -fsS -o /dev/null -w "%{http_code}\n" "https://${DASHY_HOSTNAME}/"
# Expected: 200
```

### Common first-deploy issues

- **Cert issuance fails.** DNS hasn't propagated to your server's IP yet, or port 80/443 isn't reachable from the internet. Confirm with `dig +short ${DASHY_HOSTNAME}`.
- **`docker compose up` fails with `set in .env`.** A required variable is empty in `.env`; the error names it.
- **Network not found.** Step 2 (the `docker network create` commands) was skipped.
- **Dashboard shows the starter page.** That is the shipped `config.yml`. Edit it and reload. Dashy picks up changes from the UI (Config → Update) or on container restart.

### Apply `.env` or compose-file changes

```bash
docker compose -f dashy-traefik-letsencrypt-docker-compose.yml -p dashy up -d --force-recreate
```

## Supply chain trust

This repository is a deployment template, not a custom image. It orchestrates two upstream images:

- [`lissy93/dashy`](https://hub.docker.com/r/lissy93/dashy): Dashy upstream
- [`traefik`](https://hub.docker.com/_/traefik): reverse proxy, Docker Hub official image

Both are pinned to `tag@sha256:<digest>` as interpolation defaults in the compose file's `x-images` block. Compose pulls by digest, not by tag, so two users deploying on different days get byte-identical image manifests. And `git pull` alone delivers the version combination this repository has tested. Setting `DASHY_IMAGE_TAG` or `TRAEFIK_IMAGE_TAG` in `.env` overrides the default when you deliberately want a different version.

The daily `check-pin-freshness` CI job re-resolves each pinned tag against its registry and compares the pinned Dashy version against the newest Docker Hub tag (upstream publishes patch tags there without cutting a GitHub release for each) and the pinned Traefik version against the latest upstream release. Any drift fails the run and notifies the maintainer. GitHub Actions are pinned by commit SHA with version comments; Dependabot keeps those fresh.

## Production checklist

- [ ] **Generate your own `TRAEFIK_BASIC_AUTH` hash**: never deploy the example value from a guide.
- [ ] **Treat `config.yml` as data worth backing up**: it *is* your dashboard. Keep it in your own git repo or backup rotation.
- [ ] **Verify Let's Encrypt cert issuance.** Watch `docker compose -p dashy logs traefik -f` on first start for `Adding certificate for domain(s)`.
- [ ] **Lock down the Traefik dashboard.** Basic auth is basic. Consider Traefik's `IPAllowList` middleware or not exposing the dashboard publicly at all.
- [ ] **If your dashboard links to internal services, keep Dashy internal too**: a public start page enumerates your infrastructure for anyone who finds it.

## Unattended updates

Releases are the update channel: a tag is cut only after CI has built the pinned images, booted the full stack, and passed the smoke tests. `update.sh` moves a deployment to the newest tag and nothing else:

```bash
./update.sh --dry-run   # show what would be applied
./update.sh             # update within the current major and redeploy
```

Put it on a timer for hands-off minor/patch updates:

```bash
# crontab -e
17 5 * * *  /opt/dashy-traefik-letsencrypt-docker-compose/update.sh >> /var/log/dashy-update.log 2>&1
```

The script refuses to cross a MAJOR template version on its own: majors are breaking by definition and their release notes exist to be read. After reading them, `./update.sh ‑‑allow-major` performs the jump. It also refuses to touch a checkout with local modifications: your customization belongs in `.env`, which updates never overwrite.

This is deliberately a host-side script and not a container in the stack: an in-stack updater needs the Docker socket (root on the host) and turns "someone pushed to a repo" into "someone deployed to your machine" with no operator in the loop. A cron job under your own user updates only to tagged, CI-verified states and leaves the trust boundary where it was.

## Resource limits

Every service carries memory and CPU limits plus reservations as compose-level defaults: the same values CI boots the stack under. Override any of them in `.env` (the knobs and their defaults are listed in `.env.example`, e.g. `TRAEFIK_MEMORY_LIMIT=512m`) and the override survives every `git pull`. If a service is OOM-killed under real load, `docker inspect <container> ‑‑format '{{.State.OOMKilled}}'` says so; raise its `_MEMORY_LIMIT` and recreate.

## Backups

Dashy has no database: `config.yml` next to the compose file *is* the dashboard, and Traefik's certificates live in the `traefik-certificates` volume and are re-issued automatically. Keep `config.yml` in your own git repository (it is the one file worth versioning), and there is nothing else to back up.

## Container hardening

Every service runs with `security_opt: no-new-privileges:true`, so a process cannot gain privileges through setuid binaries even if it escapes its initial capability set. Infrastructure containers (the reverse proxy, databases, caches, backups) run with `cap_drop: [ALL]` and add back only what their entrypoints need: `NET_BIND_SERVICE` for Traefik to bind :80/:443, `CHOWN`/`SETUID`/`SETGID` (and friends) for database images to own their data directory and drop to their service user. Application containers keep the default capability set on purpose: upstream images assume it, and a wrong guess there is a boot loop in production rather than a hardening win. CI boots the stack under exactly these settings on every push, so what ships is what was tested.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every day at 06:00 UTC: shellcheck + actionlint, a Trivy scan of each pinned image, the daily `check-pin-freshness` job, and a deploy-and-test job that boots the full stack with an ephemeral `.env`, requests real routing through Traefik, and requires the Dashy UI to answer 200 over HTTPS with the shipped `config.yml` mounted.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** · Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
