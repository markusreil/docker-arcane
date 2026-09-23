# Arcane

Self-hosted Docker Compose project for [Arcane](https://getarcane.app/), a
Docker management UI. Arcane runs behind an **external** reverse proxy and
manages the host Docker daemon.

## Architecture

```
              Internet
                 │
                 ▼
        ┌──────────────────┐
        │  external proxy  │   nginx-proxy + acme-companion / self-signed
        │  (already up)    │   terminates TLS (variant decides how)
        └────────┬─────────┘
                 │  shared external network: ${NGINX_PROXY_NETWORK}
                 ▼
        ┌──────────────────┐
        │  arcane          │   ghcr.io/getarcaneapp/manager
        │  expose :3552    │   no host ports published
        └────────┬─────────┘
                 │  bind mount
                 ▼
        /var/run/docker.sock        arcane-data (named volume) → /app/data
```

* **External reverse proxy (spec rule 4).** This project does not run a proxy.
  It joins the shared external proxy network and advertises its proxy contract —
  `VIRTUAL_HOST`, `VIRTUAL_PORT`, `ACME_HOST` and `GEN_SELF_SIGNED_CERT` (the
  latter wired from `ARCANE_GEN_SELF_SIGNED_CERT`, which defaults to `false` for
  an ACME-only deployment). Set `ARCANE_GEN_SELF_SIGNED_CERT=true` for a
  variant-agnostic stack that also serves the LAN/self-signed variant. The
  proxy owns all public endpoints; there are **no `ports:` mappings**.
* **Docker socket.** Arcane manages the host daemon, so it needs
  `/var/run/docker.sock` bind-mounted (the documented exception to rule 7). See
  [`arcane/README.md`](arcane/README.md) for the security reasoning.
* **State.** All persistent data lives in the named volume `arcane-data`
  (`/app/data`), not on a host path (rule 7).

## Prerequisites

* Docker Engine + Docker Compose v2.
* An external reverse proxy already running and connected to a shared network,
  e.g. [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) with
  [acme-companion](https://github.com/nginx-proxy/acme-companion) for TLS.
* The shared network named by `NGINX_PROXY_NETWORK` must already exist.
* A DNS record for `arcane.<BASE_DOMAIN>` pointing at the proxy host.

## Quickstart

```sh
cp env.example .env
# edit .env: set BASE_DOMAIN, NGINX_PROXY_NETWORK, ARCANE_VERSION and
# generate real secrets (openssl rand -hex 32)
docker compose config     # sanity check; fails fast on missing vars
docker compose up -d
docker compose ps
```

Then browse to `https://arcane.<BASE_DOMAIN>`. On **first run**, Arcane presents a
setup screen where you create the initial admin account through the web UI —
no credentials are seeded by this project.

## Operational notes

* **No host ports.** Reach the UI only through the reverse proxy at
  `https://arcane.<BASE_DOMAIN>`. Arcane is exposed to the proxy on port `3552`
  inside the shared network.
* **Secrets.** `ARCANE_ENCRYPTION_KEY` and `ARCANE_JWT_SECRET` must be
  generated and kept stable. Changing `ARCANE_ENCRYPTION_KEY` after data has
  been written makes previously encrypted secrets unreadable; changing
  `JWT_SECRET` invalidates sessions. Both live only in `.env` (gitignored).
* **Volume reseed.** Arcane has no first-run config seeding from this project
  (it is an upstream image — see the service README). If you ever need to reset
  Arcane's state, remove the service and its volume, then bring it up again:

  ```sh
  docker compose down
  docker volume rm arcane_arcane-data
  docker compose up -d
  ```

  The volume is prefixed with the project name (`arcane_`).
* **Version pinning.** Set `ARCANE_VERSION` to a specific released tag (e.g.
  `v2.13.1`) for reproducibility; avoid `latest`.
* **Homepage.** Labels are set for [Homepage](https://gethomepage.dev/)
  (`homepage.group`, `homepage.name`, `homepage.icon`, `homepage.href`,
  `homepage.description`, `homepage.showStats`). The icon (`arcane`, from the
  [dashboard-icons](https://github.com/homarr-labs/dashboard-icons) CDN) and
  description are required for the card to render non-empty; `homepage.showStats`
  (note: not `homepage.stats`) expands the container CPU/memory/network block
  and needs Homepage's Docker integration to be configured.

## Files

| File | Purpose |
| --- | --- |
| `docker-compose.yml` | Compose project (`name: arcane`), single service |
| `env.example` | Tracked example config — source of truth for variable shape |
| `.env` | Local secrets/config (gitignored; copy from `env.example`) |
| `.gitignore` | Ignores `.env` |
| `CHANGELOG.md` | Notable changes, [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) format |
| `AGENTS.md` | Ongoing rules for agents working in this repo |
| `arcane/README.md` | Service details, env vars, security reasoning |
