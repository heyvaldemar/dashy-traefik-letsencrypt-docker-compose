# Dashy + Traefik + Let's Encrypt — Docker Compose

[![Deployment Verification](https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml/badge.svg?branch=main)](https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml)

This repository deploys **Dashy 4** — a self-hosted dashboard for all your services — behind **Traefik** with automatic **Let's Encrypt TLS**. One `docker compose up` away from your own start page at `https://your-domain`.

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
- **Dashboard shows the starter page.** That is the shipped `config.yml`. Edit it and reload — Dashy picks up changes from the UI (Config → Update) or on container restart.

### Apply `.env` or compose-file changes

```bash
docker compose -f dashy-traefik-letsencrypt-docker-compose.yml -p dashy up -d --force-recreate
```

## Supply chain trust

This repository is a deployment template, not a custom image. It orchestrates two upstream images:

- [`lissy93/dashy`](https://hub.docker.com/r/lissy93/dashy) — Dashy upstream
- [`traefik`](https://hub.docker.com/_/traefik) — reverse proxy, Docker Hub official image

Both are pinned to `tag@sha256:<digest>` as interpolation defaults in the compose file's `x-images` block. Compose pulls by digest, not by tag, so two users deploying on different days get byte-identical image manifests — and `git pull` alone delivers the version combination this repository has tested. Setting `DASHY_IMAGE_TAG` or `TRAEFIK_IMAGE_TAG` in `.env` overrides the default when you deliberately want a different version.

The daily `check-pin-freshness` CI job re-resolves each pinned tag against its registry and compares the pinned Dashy version against the newest Docker Hub tag (upstream publishes patch tags there without cutting a GitHub release for each) and the pinned Traefik version against the latest upstream release — any drift fails the run and notifies the maintainer. GitHub Actions are pinned by commit SHA with version comments; Dependabot keeps those fresh.

## Production checklist

- [ ] **Generate your own `TRAEFIK_BASIC_AUTH` hash** — never deploy the example value from a guide.
- [ ] **Treat `config.yml` as data worth backing up** — it *is* your dashboard. Keep it in your own git repo or backup rotation.
- [ ] **Verify Let's Encrypt cert issuance.** Watch `docker compose -p dashy logs traefik -f` on first start for `Adding certificate for domain(s)`.
- [ ] **Lock down the Traefik dashboard.** Basic auth is basic. Consider Traefik's `IPAllowList` middleware or not exposing the dashboard publicly at all.
- [ ] **If your dashboard links to internal services, keep Dashy internal too** — a public start page enumerates your infrastructure for anyone who finds it.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every day at 06:00 UTC: shellcheck + actionlint, a Trivy scan of each pinned image, the daily `check-pin-freshness` job, and a deploy-and-test job that boots the full stack with an ephemeral `.env`, requests real routing through Traefik, and requires the Dashy UI to answer 200 over HTTPS with the shipped `config.yml` mounted.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** — Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
