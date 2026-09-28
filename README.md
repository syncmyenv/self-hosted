# SyncMyEnv — self-hosted

Run your own [SyncMyEnv](https://syncmyenv.com) server: the same code as SyncMyEnv Cloud,
on your hardware. It stores **ciphertext only** — your `.env` files are encrypted on your
machines before anything is uploaded, and the server can't read or forge them
([how](https://github.com/syncmyenv/core/blob/main/docs/sync.md)).

- One small container (≈25 MB, distroless, non-root, read-only filesystem)
- One SQLite file for data — trivial to back up
- `amd64` + `arm64` (Raspberry Pi, NAS, cheap VPS)

## Install (with automatic HTTPS)

You need Docker, a hostname pointing at the machine (DNS `A`/`AAAA` record) and ports
**80 + 443** open. [Caddy](https://caddyserver.com) gets a Let's Encrypt certificate for you.

```bash
git clone https://github.com/syncmyenv/self-hosted syncmyenv && cd syncmyenv
cp .env.example .env
$EDITOR .env                        # SME_DOMAIN=env.example.com
docker compose up -d
docker compose logs server          # ← first-run setup code
```

Open `https://env.example.com/setup` and create the admin account with the **setup code**
from the log. The code protects a freshly started server from whoever finds it first;
it's only printed until an admin exists.

## Connect your machines

```bash
brew install syncmyenv/tap/sme      # or: curl -fsSL https://syncmyenv.com/install.sh | sh
sme login env.example.com           # approve in the browser
sme sync                            # first time asks your vault password once
sme daemon install                  # every save is sealed and pushed automatically
```

New machine: `sme login env.example.com` → `sme sync` → `sme restore --missing`.

## Already running a reverse proxy?

Traefik, nginx, Cloudflare Tunnel, Tailscale Serve… use the proxy-less variant, which
binds to `127.0.0.1:8080` only:

```bash
docker compose -f compose.behind-proxy.yaml up -d
```

Serve it over **HTTPS** (`sme login` refuses plain `http://` for public hostnames) and
forward `X-Forwarded-For`. Keep port 8080 reachable by the proxy alone.

## Upgrade

```bash
docker compose pull && docker compose up -d
```

Pin `SME_VERSION` in `.env` for predictable upgrades. Database migrations run automatically.
Back up first (below) — it takes a second.

## Backup & restore

The whole server state is one SQLite database in the `data` volume. Backups are
**consistent while the server runs** (SQLite online backup, not a file copy):

```bash
docker compose exec -T server /server backup - > syncmyenv-$(date +%F).db
```

Restore (e.g. onto a new machine):

```bash
docker compose stop server
docker compose run --rm -T server restore - < syncmyenv-2026-09-28.db
docker compose start server
```

`restore` checks the backup's integrity first, refuses to run while the server is up,
and keeps the previous database as `syncmyenv.db.before-restore-<time>`.

The backup contains only ciphertext plus account metadata (emails, device names).
Treat it like any other server backup; it's useless without your users' vault passwords.

## Configuration

| Variable | Default | |
|---|---|---|
| `SME_DOMAIN` | — | public hostname (required) |
| `SME_VERSION` | `latest` | image tag to run |
| `SME_BASE_URL` | `https://$SME_DOMAIN` | full public URL, if different |
| `SME_HTTP_PORT` / `SME_HTTPS_PORT` | `80` / `443` | host ports for Caddy |
| `SME_PORT`, `SME_BIND` | `8080`, `127.0.0.1` | behind-proxy variant only |

Advanced (set under `server.environment`): `SME_SETUP_CODE` to choose the first-run code,
`SME_ALLOW_SETUP=0` to disable first-run setup entirely.

## Security notes

- Keep the server updated; watch [releases](https://github.com/syncmyenv/core/releases).
- The image is signed with cosign:
  `cosign verify ghcr.io/syncmyenv/server:latest --certificate-identity-regexp '^https://github.com/syncmyenv/server/' --certificate-oidc-issuer https://token.actions.githubusercontent.com`
- Report vulnerabilities privately:
  [github.com/syncmyenv/core/security](https://github.com/syncmyenv/core/security/advisories/new).

## Uninstall

```bash
docker compose down        # stop (keeps data)
docker compose down -v     # stop and DELETE all data — back up first
```

---

The deployment files in this repo are Apache-2.0. The server image is published by SyncMyEnv;
the CLI and sync protocol are open source at [syncmyenv/core](https://github.com/syncmyenv/core).
