# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

_(no unreleased changes yet)_

## [2.1.0] - 2026-09-02

### Added

- **`update.sh`** — unattended updates to the newest tagged release,
  and nothing else: a tag is cut only after CI has booted the pinned
  images and passed the smoke tests, so "update to the latest tag" means
  "update to a combination a machine has already run". It refuses to
  cross a major version on its own (`--allow-major` after reading the
  notes), refuses a checkout with local modifications, and supports
  `--dry-run`. Put it on a cron timer for hands-off minor/patch updates.

## [2.0.0] - 2026-09-02

### Changed (major: Dashy 3.x → 4.x)

- **Dashy updated to 4.6.7** (was release-3.1.15). The 4.x line is the
  active upstream line; 3.x receives no further releases. The shipped
  starter `config.yml` works unchanged, and in testing a 3.x-era config
  loaded cleanly — still, skim your own `config.yml` against the
  [Dashy 4 docs](https://dashy.to/docs/) after upgrading.
- **Config mount moved to `/app/user-data/conf.yml`** — the path Dashy 4
  reads. If you override the volume in your own compose file, update the
  container-side path.
- **`platform: linux/amd64` removed**: 4.x images are multi-arch
  (amd64 + arm64), so ARM servers now run natively instead of under
  emulation.
- The freshness gate reads the newest Dashy version from Docker Hub tags
  instead of GitHub releases — upstream publishes patch tags (such as
  4.6.7) without cutting a GitHub release for each.

## [1.0.0] - 2026-09-02

First semver release. Brings this template to the fleet standard established
in [keycloak-traefik-letsencrypt-docker-compose](https://github.com/heyvaldemar/keycloak-traefik-letsencrypt-docker-compose).

### Changed

- **Dashy updated to release-3.1.15** (was release-3.1.1) and **Traefik to
  v3.7** (was 3.2), both pinned by `tag@sha256:digest` in the compose
  `x-images` block. `git pull` delivers the tested versions;
  `DASHY_IMAGE_TAG` / `TRAEFIK_IMAGE_TAG` in `.env` override deliberately.
- Required variables (`DASHY_HOSTNAME`, `TRAEFIK_ACME_EMAIL`,
  `TRAEFIK_HOSTNAME`, `TRAEFIK_BASIC_AUTH`) now fail fast with a
  `${VAR:?…}` guard instead of silently deploying half-configured;
  `TRAEFIK_LOG_LEVEL` has a default.
- `.env` is no longer tracked in git. Copy `.env.example` to `.env` and
  fill in your values; the previous tracked file carried only example
  values, so nothing needs rotating.
- Removed the unused `DASHY_URL` variable.

### Added

- **Deployment Verification workflow**: shellcheck + actionlint; a Trivy
  scan of each pinned image; daily `check-pin-freshness` (digest drift +
  Dashy/Traefik release lag); and a deploy-and-test job that boots the
  full stack with an ephemeral `.env` and requires the Dashy UI to answer
  200 over HTTPS through Traefik.
- `.env.example` with generation commands; `.gitignore` for `.env`.

[Unreleased]: https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/compare/v2.1.0...HEAD
[2.1.0]: https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/compare/v2.0.0...v2.1.0
[2.0.0]: https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/compare/v1.0.0...v2.0.0
[1.0.0]: https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/releases/tag/v1.0.0
