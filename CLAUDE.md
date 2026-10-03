# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

magnusfroste's **thin Easypanel deployment repo** for upstream
[`pewdiepie-archdaemon/odysseus`](https://github.com/pewdiepie-archdaemon/odysseus). It contains
deployment config only — no application source, no Dockerfile, no build step. It replaced a full
source fork (previously `odysseus-easy`, archived) whose source tree was dead weight: deploys pull
the official prebuilt image instead.

- `docker-compose.yml` — the entire deployment: `odysseus` pulls
  `ghcr.io/pewdiepie-archdaemon/odysseus:${ODYSSEUS_TAG:-latest}` (`pull_policy: always`; `:latest`
  = upstream `main`, `:dev` = upstream `dev`, `1.0.0-dev.<sha>` = pinned) plus `chromadb`,
  `searxng`, and `ntfy`. Fork-specific tweak: odysseus joins the external `easypanel` network with
  alias `app_odysseus_odysseus` for Traefik routing — keep that intact. No `ports:` on odysseus,
  only `expose:` — a host-port binding makes a second instance fail with "port is already
  allocated" while Easypanel reports success (hermes-easy lesson). All odysseus state lives in
  named volumes (`odysseus-data` etc.), never bind mounts in the checkout, which a re-clone or
  service recreate wipes. An `env_file: .env (required: false)` passthrough hands every Easypanel
  panel variable to the container; the explicit `environment:` list wins for the same name.
- `config/searxng/settings.yml` — template mounted into searxng; `__SEARXNG_SECRET__` is replaced
  at container boot by the inline entrypoint wrapper in the compose file.
- `scripts/migrate_searxng_settings.py` — vendored verbatim from upstream; the searxng entrypoint
  wrapper runs it (advisory, `|| true`) so a settings.yml retained in the volume inherits new
  defaults across searxng upgrades. When re-syncing, copy upstream's file as-is.
- `.env.example` — documents every env var the compose file consumes. Real values live in
  Easypanel's env panel, which writes them to `.env` next to the compose file. Never commit `.env`.

App features belong upstream, not here. Changes here should only adjust deployment wiring.

## Commands

```bash
docker compose config                      # validate after editing the compose file
docker compose up -d                       # pulls images; nothing builds locally
docker compose logs --tail=120 odysseus
```

Upgrading the app = change `ODYSSEUS_TAG` (or nothing, for `latest`) and redeploy; the tag is
re-pulled every time.

## Constraints worth knowing

- **searxng is pinned, not `:latest`** — odysseus's startup waits on searxng's healthcheck
  (`depends_on: condition: service_healthy`), so a broken upstream searxng tag blocks the whole
  app. Bump the pin only after verifying the new tag boots clean.
- The docker.sock mount + `group_add: ${DOCKER_GID}` + `ODYSSEUS_ENABLE_HOST_DOCKER=true` let
  Odysseus's Cookbook run `docker exec` against sibling containers (Ollama etc.). The GID must
  match the host's docker group. This is a deliberate divergence: upstream ships the socket as an
  opt-in overlay (`docker/host-docker.yml`) and the app ignores the socket without the flag —
  keep mount and flag together.
- Ports for chromadb/searxng/ntfy bind host loopback; odysseus publishes nothing and is reached
  through Traefik via the `easypanel` network.

## Syncing against upstream

The image self-updates, but the config surface drifts. To re-sync (last done 2026-10-03):
fetch upstream `main`'s `docker-compose.yml` and `.env.example` from
`raw.githubusercontent.com/pewdiepie-archdaemon/odysseus/main/...`, diff the `environment:`
blocks and env-var lists against this repo's, and port over new vars, default changes, the
searxng pin, and `scripts/migrate_searxng_settings.py` (copy verbatim). Expected intentional
diffs that must survive a sync: `image:` instead of `build: .`, `expose` instead of `ports`,
named volumes instead of bind mounts, the `easypanel` network block, `env_file` passthrough,
and the baked-in docker.sock + `ODYSSEUS_ENABLE_HOST_DOCKER` pair.
