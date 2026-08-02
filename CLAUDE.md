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
  alias `app_odysseus_odysseus` for Traefik routing — keep that intact.
- `config/searxng/settings.yml` — template mounted into searxng; `__SEARXNG_SECRET__` is replaced
  at container boot by the inline entrypoint wrapper in the compose file.
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
- The docker.sock mount + `group_add: ${DOCKER_GID}` let Odysseus's Cookbook run `docker exec`
  against sibling containers (Ollama etc.). The GID must match the host's docker group.
- Ports for chromadb/searxng/ntfy bind loopback by default; only odysseus is meant to be reached
  through Traefik via the `easypanel` network.
