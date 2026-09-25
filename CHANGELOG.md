# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Ephemeral `tmpfs` at `/builds` on the Arcane service so its Build Workspace
  ("Container Images for local use or push to a registry") has the writable
  directory it expects without persisting temporary build contexts.
- `homepage.showStats` label on the Arcane service so its Homepage card can
  expand the container CPU/memory/network stats block.
- `AGENTS.md` summarizing the ongoing project rules for agents working in the repo.
- `CHANGELOG.md` following the Keep a Changelog format.

### Changed

- Set the `arcane-url` `x-hosts` anchor to `https://arcane.<BASE_DOMAIN>` so
  `APP_URL` and `homepage.href` match the TLS-terminated proxy URL (was
  `http://`); Arcane derives CORS origins, OIDC redirect URIs and WebAuthn
  (passkey) origins from `APP_URL`, so the scheme must be `https` behind the
  proxy.
- Replaced the service-specific `ARCANE_HOST` variable with the cluster-wide
  `BASE_DOMAIN`; Arcane is now served at `arcane.<BASE_DOMAIN>`, anchored in
  `x-hosts` and referenced by `VIRTUAL_HOST`, `ACME_HOST` and `APP_URL`.
- Bumped the pinned Arcane image from `v2.6.0` to `v2.13.1`; the old pin only
  supported SQLite schema 68 and failed against a database already migrated to
  schema 87. Startup now forward-migrates the database automatically.
- Renamed the compose network to `web-proxy` and defaulted `NGINX_PROXY_NETWORK`
  to `web-proxy` (`${NGINX_PROXY_NETWORK:-web-proxy}`).
- Made the proxy TLS contract variant-agnostic: replaced the deprecated
  `LETSENCRYPT_HOST` with `ACME_HOST` and added the `GEN_SELF_SIGNED_CERT`
  opt-in, wired from the new optional `ARCANE_GEN_SELF_SIGNED_CERT` variable
  with a `false` default (set `true` for a LAN/self-signed deployment or local
  testing).
- Sourced `APP_URL` and `homepage.href` from `x-hosts` anchors instead of
  re-interpolating `ARCANE_HOST`.

### Deprecated

### Removed

### Fixed

- `failed to ensure builds directory: mkdir /builds: permission denied` —
  Arcane drops to its unprivileged runtime UID (`65532`) and `/builds` did not
  exist; the service now provides a writable `/builds` via an ephemeral
  `tmpfs`.
- Added the missing `homepage.icon` (`arcane`) and `homepage.description` labels
  so the Arcane card renders instead of appearing empty in Homepage.

### Security
