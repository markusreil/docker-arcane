# arcane (service)

Arcane — a Docker management UI — running from the upstream image.

## Why the upstream image (spec rule 6)

Arcane is a complete third-party application distributed as an upstream image,
`ghcr.io/getarcaneapp/manager`. The upstream image **is** the service: it ships
the application, its HTTP server on port `3552`, its own user handling and its
own data layout. Rebuilding it here would mean re-packaging someone else's
release with no benefit, so this project consumes the image as-is and pins the
tag via `ARCANE_VERSION`.

## Why there is no Dockerfile or entrypoint.sh (spec rules 8-9)

* **No `Dockerfile`** — there is nothing to build (rule 6). The image tag is
  selected from `.env` and pulled at deploy time.
* **No `entrypoint.sh`** — entrypoints in this repo are *write-once config
  seeders* for custom-built images (rule 8). Arcane requires no first-run file
  seeding: it initializes its own data directory on first start and exposes a
  web-based setup wizard for the initial admin account. Adding an entrypoint
  would only risk overwriting state Arcane manages itself.
* **No `PUID`/`PGID` drop** — rule 9 applies only where a custom entrypoint
  exists. The upstream image manages its own user and file ownership. Run it
  as-is.

Because no seeding happens, there is no reseed-on-first-start logic to
preserve. The reset procedure (below) is purely operational.

## Environment variables

| Variable | Required | Default | Description |
| --- | --- | --- | --- |
| `ARCANE_VERSION` | yes | — | Upstream image tag, e.g. `v2.13.1`. Pin for reproducibility. |
| `BASE_DOMAIN` | yes | — | Cluster base domain. Arcane is served at `arcane.<BASE_DOMAIN>`, anchored in `x-hosts` and reused for `VIRTUAL_HOST`, `ACME_HOST`, the `GEN_SELF_SIGNED_CERT` opt-in and `APP_URL`. |
| `ARCANE_GEN_SELF_SIGNED_CERT` | no | `false` | Self-signed TLS opt-in for the LAN proxy variant. `true` requests a self-signed cert for `arcane.<BASE_DOMAIN>`; `false` relies on the ACME opt-in only. Wired to `GEN_SELF_SIGNED_CERT`. |
| `ARCANE_ENCRYPTION_KEY` | yes | — | 32-byte key (raw/base64/hex) used to encrypt stored secrets. Generate: `openssl rand -hex 32`. Must stay stable. |
| `ARCANE_JWT_SECRET` | yes | — | Secret used to sign JWT session tokens. Generate: `openssl rand -hex 32`. |
| `ARCANE_TRUSTED_PROXIES` | no | *(empty)* | Comma-separated proxy CIDRs/IPs trusted to set `X-Forwarded-*`. Empty = trust none. |
| `NGINX_PROXY_NETWORK` | no | `web-proxy` | Name of the pre-existing external proxy network this service joins. |
| `TZ` | no | `UTC` | Container timezone (IANA name). |

Proxy integration vars set by the compose file (not in `.env`): `VIRTUAL_HOST`,
`VIRTUAL_PORT=3552`, `ACME_HOST`, `APP_URL=https://<host>`; `GEN_SELF_SIGNED_CERT`
is wired from the `ARCANE_GEN_SELF_SIGNED_CERT` variable in `.env`.

### Proxy contract (spec rule 4)

The service declares its proxy contract through `.env`:

* `ACME_HOST` — real-certificate opt-in, honoured only by the internet-facing
  variant (where `acme-companion` runs). Inert elsewhere.
* `GEN_SELF_SIGNED_CERT` — self-signed opt-in, honoured only by the LAN variant
  (where the local cert generator runs). Inert elsewhere. Driven by
  `ARCANE_GEN_SELF_SIGNED_CERT`, which **defaults to `false`** (ACME-only /
  internet-facing deployment); override it to `true` for a LAN/self-signed
  deployment or local testing.

The compose file wires this opt-in with `${ARCANE_GEN_SELF_SIGNED_CERT:-false}`,
so the key is always declared and the same compose file deploys unchanged
against every variant; only `.env` differs. Set `ARCANE_GEN_SELF_SIGNED_CERT=true`
to keep both opt-ins enabled (each cluster variant honours exactly one and
ignores the other).

`arcane.<BASE_DOMAIN>` is an exact, filename-safe hostname (given a plain
`BASE_DOMAIN`), so it can become a `<host>.crt` file for the self-signed variant.
No `HTTPS_METHOD`, `HSTS`,
`CERT_NAME`, per-service contact email or `ports:` are set — TLS behaviour is
cluster policy owned by the proxy. The older `LETSENCRYPT_HOST` spelling is
deprecated and not used; only the `LETSENCRYPT_TEST` staging toggle keeps its
name.

## Docker socket bind mount — security reasoning (rule 7 exception)

The compose file bind-mounts `/var/run/docker.sock:/var/run/docker.sock`. This
is the documented special-case exception to the "no stateful bind mounts" rule
(rule 7): a Unix socket is a special device, not stateful data.

Arcane's entire purpose is managing containers, images, networks and volumes on
the host, which it does through the Docker API exposed on that socket. There is
no read-only mode that still provides management functionality.

Security implications to be aware of:

* Access to the Docker socket is effectively **root-equivalent** on the host:
  anyone who can reach the socket can start privileged containers and mount the
  host filesystem. Treat Arcane as a highly privileged administrative tool.
* Do **not** publish Arcane to the public internet without TLS and
  authentication. This project puts it behind the external proxy (TLS provided
  by the cluster — ACME or self-signed depending on the variant) and relies on
  Arcane's own login. Restrict access at the proxy/network layer if the host is
  internet-facing.
* Set `ARCANE_TRUSTED_PROXIES` to the proxy's address so forwarded headers are
  only honored from the trusted proxy.

## Persistent data and reseed procedure (rule 7)

State lives in the named volume `arcane-data` mounted at `/app/data` (standard
named volume, no host bind mount).

To reset Arcane's state (e.g. start over):

```sh
docker compose down
docker volume rm arcane_arcane-data   # project-name-prefixed
docker compose up -d
```

Removing only the container (without the volume) preserves all data. Note that
the volume name is prefixed with the Compose project name `arcane` (set via
`name:` in `docker-compose.yml`).
