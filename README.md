# odysseus-easy

Thin Easypanel deployment for [Odysseus](https://github.com/pewdiepie-archdaemon/odysseus),
a self-hosted AI workspace (chat, agents, research, documents, email, notes, calendar,
local model workflows).

No source code lives here and nothing is built here. `docker-compose.yml` pulls the official
prebuilt image `ghcr.io/pewdiepie-archdaemon/odysseus:${ODYSSEUS_TAG:-latest}` with
`pull_policy: always`, so every Easypanel redeploy fetches the newest upstream build.
This repo replaces a full source fork that previously had to be kept in sync with upstream —
now there is nothing to sync, ever.

## What's in the box

| File | Purpose |
|---|---|
| `docker-compose.yml` | The whole deployment: odysseus + chromadb (vectors) + searxng (web search) + ntfy (notifications). The odysseus service joins the external `easypanel` network with alias `app_odysseus_odysseus` so Traefik can route to it. |
| `config/searxng/settings.yml` | Template mounted into the searxng container (secret injected at boot). |
| `.env.example` | All env vars the compose file consumes, ready to paste into Easypanel. |

## Easypanel setup

1. **Create a service** → Compose → source **Git**: this repo, branch `main`,
   compose file `docker-compose.yml`.
2. **Env panel**: paste what you need from `.env.example`. Minimum for a public deploy:
   `ODYSSEUS_ADMIN_PASSWORD`, `ALLOWED_ORIGINS` (your public origin), `SECURE_COOKIES=true`.
3. **Domain** → point it at the odysseus service, port `7000` (routing goes via the
   `easypanel` network alias).
4. **Deploy.** First boot prints the admin password in the odysseus logs if you didn't
   pre-seed one.

## Choosing / upgrading the version

Set `ODYSSEUS_TAG` in the env panel:

- `latest` (default) — upstream `main`, the curated branch
- `dev` — upstream's newest changes
- `1.0.0-dev.<sha>` — pin an exact build

Because of `pull_policy: always`, plain **Redeploy** always re-pulls the tag — that's the
whole upgrade procedure.

## Notes

- **searxng is pinned** (not `:latest`): odysseus waits on searxng's healthcheck, so a broken
  upstream searxng tag would block the whole app from starting. Bump the pin deliberately
  after verifying a newer tag boots clean.
- The docker.sock mount + `DOCKER_GID` exist so Odysseus's Cookbook can reach sibling
  containers (e.g. Ollama). Remove both if you don't use that.
