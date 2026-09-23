# AGENTS.md

Ongoing rules for agents working in this repo. This is a summary, not the full
spec — see [`README.md`](README.md) and [`arcane/README.md`](arcane/README.md)
for detail. Keep this file in sync when the project's rules change.

## Configuration

* All deployment configuration lives in `.env` (hidden, never tracked). The
  tracked source of truth is `env.example`; copy it with `cp env.example .env`.
* Keep `env.example` and `.env` in the same order and shape: global vars at the
  top, then one `# --- <service> ---` section per service, one comment per var.
* Required vars use fail-fast `${VAR:?...}` interpolation — never substitute a
  silent default for required configuration. Optional vars use `${VAR:-default}`
  explicitly and are documented as optional.
* Never scatter secrets into other files; `.env` stays gitignored.

## Reverse proxy integration

* Never publish `ports:` — the proxy owns all public endpoints. Advertise the
  app with `expose:`.
* Join the shared external proxy network (`NGINX_PROXY_NETWORK`, declared
  `external: true`, same name as the cluster's; default `web-proxy`) and start
  the service **after** the proxy cluster is up.
* Declare the **complete** contract, variant-agnostic — the proxy fails
  silently, not loudly, on anything omitted:
  * `VIRTUAL_HOST` — always.
  * `VIRTUAL_PORT` / `VIRTUAL_PROTO` — where the defaults do not fit.
  * For HTTPS, declare `ACME_HOST` plus the `GEN_SELF_SIGNED_CERT` opt-in. The
    self-signed opt-in is `false` by default; expose it via an optional `.env`
    variable (`${<SERVICE>_GEN_SELF_SIGNED_CERT:-false}`) so local testing can
    override it to `true` without editing the compose file. Each variant honours
    exactly one opt-in; omitting the key takes that choice away. In this repo it
    is wired from `ARCANE_GEN_SELF_SIGNED_CERT` in `.env`.
* Use the `ACME_*` spelling for every ACME variable; the deprecated
  `LETSENCRYPT_*` aliases must not be used (only `LETSENCRYPT_TEST` keeps its
  name). Do not set `HTTPS_METHOD`, `HSTS`, `CERT_NAME`, or a per-service
  contact email unless deliberately overriding cluster policy.
* Self-signed hosts must be exact, filename-safe names (no wildcard/regexp/
  port/path), since such values are silently skipped.
* Define `x-hosts` anchors at the top of `docker-compose.yml` and reference them
  everywhere the hostname is needed (`VIRTUAL_HOST`, `ACME_HOST`, app-level
  hostname settings) — never copy-paste literals.

## State, naming, runtime

* Use standard named volumes for all stateful data. Host bind mounts are only
  acceptable for documented special cases (Docker socket, devices) or config
  overlays, with the reasoning recorded in the service README.
* Set the project name once (`name:` at the top of the compose file). Never use
  `container_name:` — rely on default container naming.
* `restart: unless-stopped` by default.
* Set Homepage labels (`homepage.group`, `homepage.name`, `homepage.icon`,
  `homepage.href`, `homepage.description`); without `icon`/`description` the
  card renders visually empty. The stats block is `homepage.showStats` (not
  `homepage.stats`).
* Custom-built images only: `entrypoint.sh` is a write-once config seeder
  (first start only, never overwrites) and drops privileges via `PUID`/`PGID`
  (create user/group, chown config dirs recursively, chown data dirs at the
  root only, then `su-exec`). Upstream images with no seeding need no
  entrypoint — document why in the service README.

## Changelog

* Update `CHANGELOG.md` **as changes are made** — do not batch entries.
* Add entries under the matching `## [Unreleased]` subsection (`Added`,
  `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`), newest first,
  ISO-8601 dates.
* Do not create a release section on your own; only when the user asks for a
  release.

## Verification

```sh
cp env.example .env      # .env is never committed
docker compose config    # fails fast on missing required vars
docker compose build
docker compose up -d
docker compose ps
```
