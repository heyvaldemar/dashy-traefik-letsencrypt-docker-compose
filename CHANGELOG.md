# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

_(no unreleased changes yet)_

## [2.4.1] - 2026-09-07

### Changed

- **`update.sh` names any new required variable before it moves.** An update can add a required variable; `docker compose up` used to stop on it after the checkout, with the tree already on the new tag. The script now lists the variables that appeared in `.env.example` since your version and refuses, before anything has moved, when a required one is not in your `.env`. Names only, never values.
- **`lissy93/dashy:4.6.9` moved to `lissy93/dashy:4.6.13`** (automated: the freshness check reported the lag, the deploy job booted the stack on the new image before this landed).

le
  reference as before. A deployment that sets neither is unchanged. The
  freshness job, the Trivy matrix and the fleet digest automation resolve
  the nested default before reading a pin. Needs Docker Compose v2.5 or
  newer (2022): v2.0 to v2.4 leave the inner `${...}` unexpanded and
  `docker compose up` fails with an invalid reference instead of
  deploying something unexpected.

### Changed

- `lissy93/dashy` 4.6.7 to 4.6.9.

## [2.3.0] - 2026-09-02

### Security

- **Container hardening.** Every service runs with
  `security_opt: no-new-privileges:true` (no privilege escalation via
  setuid binaries even if a process escapes its initial capability
  set). Infrastructure containers (the reverse proxy, databases,
  caches, backups) drop every Linux capability and add back only what
  their entrypoints need (bind :80/:443, chown a data directory, drop to
  the service user). Application containers keep the default capability
  set: upstream images assume it, and a wrong guess there is a boot loop
  in production, not a hardening win. CI boots the stack under these
  settings on every push.

## [2.2.0] - 2026-09-02

### Added

- **Resource limits on every service, as `.env`-overridable defaults.**
  Each service now carries memory and CPU limits plus reservations
  (`<SERVICE>_MEMORY_LIMIT`, `_CPU_LIMIT`, `_MEMORY_RESERVATION`,
  `_CPU_RESERVATION`, defaults listed in `.env.example`). Set any of
  them in `.env` and the override survives every `git pull`. The
  defaults are what CI boots the stack under, so they are known to be
  enough for a fresh install; raise a limit if a service is OOM-killed
  under your real load (`docker inspect` shows `OOMKilled=true`).

## [2.1.0] - 2026-09-02

### Added

- **`update.sh`**: unattended updates to the newest tagged release,
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
  loaded cleanly. Still, skim your own `config.yml` against the
  [Dashy 4 docs](https://dashy.to/docs/) after upgrading.
- **Config mount moved to `/app/user-data/conf.yml`**: the path Dashy 4
  reads. If you override the volume in your own compose file, update the
  container-side path.
- **`platform: linux/amd64` removed**: 4.x images are multi-arch
  (amd64 + arm64), so ARM servers now run natively instead of under
  emulation.
- The freshness gate reads the newest Dashy version from Docker Hub tags
  instead of GitHub releases. Upstream publishes patch tags (such as
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

[Unreleased]: https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/compare/v2.4.1...HEAD
[2.4.1]: https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/compare/v2.4.0...v2.4.1
[2.4.0]: https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/compare/v2.3.0...v2.4.0
[2.3.0]: https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/compare/v2.2.0...v2.3.0
[2.2.0]: https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/compare/v2.1.0...v2.2.0
[2.1.0]: https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/compare/v2.0.0...v2.1.0
[2.0.0]: https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/compare/v1.0.0...v2.0.0
[1.0.0]: https://github.com/heyvaldemar/dashy-traefik-letsencrypt-docker-compose/releases/tag/v1.0.0
