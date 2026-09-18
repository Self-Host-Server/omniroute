# omniroute

Self-hosted [OmniRoute](https://github.com/diegosouzapw/OmniRoute) (free-model AI gateway) via Docker Compose, with linting, CI, and release automation wired in from [Self-Hosting-Template](https://github.com/Self-Host-Server/Self-Hosting-Template).

This runs the gateway centrally on one machine; other machines connect to it (`omniroute connect http://<this-host-ip>:20128 --api-key <key>`) instead of each running their own instance.

## Deploying

1. Copy `.env.example` to `.env` and adjust as needed — the defaults are already set up for the multi-machine use case (`REQUIRE_API_KEY=true`, `APP_BIND_HOST=0.0.0.0`, `API_HOST=0.0.0.0`, `LIVE_WS_HOST=0.0.0.0`). See the comments in `.env.example` and the header of `compose.yml` for why all four of those need to move together — verified against upstream source, not just its docs, that `API_HOST`/`LIVE_WS_HOST` are separate container-internal bind addresses from the Docker-level `APP_BIND_HOST` publish, and both layers need widening for a remote `omniroute connect` to actually reach the API/WS ports (the dashboard alone works either way, its bind address is baked into the image).
2. Start the stack:

   ```bash
   docker compose up -d
   ```

3. `omniroute` listens on:
   - `20128` — dashboard (also `PORT`)
   - `20129` — API
   - `20132` — live WebSocket

   `redis` is a required sidecar (rate limiter/cache backend) — no port published by default, since it has no `requirepass` and the app reaches it over the compose network.

4. Note the container's `mem_limit: 10g` (`OMNIROUTE_CONTAINER_MEM` in `.env.example`) — the image's own default heap (1024 MB) is documented upstream to hit a `FATAL ERROR` under real coding-agent workloads (long contexts, e.g. Claude Code). Make sure the host has the headroom before bringing this up; it will contend with anything else running on the same host otherwise.

See the comments in [`compose.yml`](compose.yml) for the full verified reasoning — none of it is copied from upstream's own `docker-compose.yml` blind, since that file builds from source and includes several optional profiles/sidecars (web, cli, host, memory, bifrost, cliproxyapi, codex-app-server) this deployment doesn't use.

## Obsidian

The `obsidian` service (linuxserver.io image) hosts the vault OmniRoute reads and writes through the [Local REST API](https://github.com/coddingtonbear/obsidian-local-rest-api) plugin. Three settings have to be changed from their defaults before OmniRoute can reach it — all three, or the connection fails identically each time:

1. **In the plugin (Obsidian → Settings → Local REST API):**
   - Turn on **Enable HTTP server**. Only the HTTPS listener on `27124` runs out of the box, and pointing OmniRoute at `27124` is not a workaround — its settings endpoint rejects that port explicitly, since the REST API it speaks is the plain-HTTP one on `27123`.
   - Set the **binding host** to `0.0.0.0`. Until you set one the plugin stores no binding host at all, and both listeners bind to the `obsidian` container's own loopback — so a request from the `omniroute` container is refused no matter what hostname it uses.
   - Copy the plugin's API key.

   These two toggles are independent, and the second is the one that usually bites: the HTTP server can be **on** — `27123` genuinely listening — and still be reachable from nowhere but inside that container. Check what the listeners are actually bound to rather than trusting the toggle:

   ```bash
   docker exec obsidian ss -lnt
   ```

   `127.0.0.1:27123` means the binding host still needs changing; `0.0.0.0:27123` is what you want. Change it in the GUI (the linuxserver image serves it on port `3000`) rather than by editing the plugin's `data.json` — Obsidian is running and will rewrite that file from memory, silently discarding the edit.

2. **In OmniRoute (dashboard → endpoint → Obsidian, or `POST /api/settings/obsidian` with `{"token": "...", "baseUrl": "..."}`):** paste that API key and set the base URL to

   ```text
   http://obsidian:27123
   ```

   **Not** `http://127.0.0.1:27123`, which is what upstream defaults to (`DEFAULT_OBSIDIAN_BASE_URL` in `src/lib/obsidian/api.ts`). Inside the `omniroute` container that loopback is that container's own — the plugin is in a different container — so the default produces:

   ```text
   [ProxyFetch] ... connect ECONNREFUSED 127.0.0.1:27123
   ```

   `compose.yml` deliberately publishes no host port for `27123`/`27124` — publishing one would do nothing for container-to-container traffic anyway, and once the binding host is `0.0.0.0` it would expose the vault's REST API on every interface behind only the plugin API key. Both services share the compose file's default network, so the service name `obsidian` resolves by DNS and is the address to use.

   If you use the API rather than the dashboard, **send `baseUrl` in the same request as the token**. Upstream only persists the URL when that field is present in the body (`setObsidianBaseUrl` is called under `if (parsed.data.baseUrl)`); POST the token alone and the route validates against the _existing_ stored value — still the `127.0.0.1` default — then reports `connected: true` without having changed anything.

This base URL is **not** configurable by env var — OmniRoute stores it in its own settings store (`src/lib/db/obsidian.ts`, namespace `obsidian`, key `base_url`), which lives in the `omniroute-data` volume, so it survives restarts but has to be set once through the dashboard or API. `.env.example` documents this where the variable would otherwise go.

Verify from inside the gateway container — this is the exact path OmniRoute uses, so it isolates the network half from the settings half:

```bash
docker exec omniroute getent hosts obsidian   # compose DNS: expect 172.x.x.x obsidian
docker exec omniroute node -e "fetch('http://obsidian:27123/').then(r=>console.log(r.status)).catch(e=>console.log('FAIL',e.cause?.code))"
```

A resolving hostname plus `FAIL ECONNREFUSED` means the network is fine and the binding host is still loopback — step 1, not step 2. `ECONNREFUSED 127.0.0.1:27123` in the gateway's own logs means the opposite: the plugin may be reachable, but OmniRoute never got told the URL, so it is still using `DEFAULT_OBSIDIAN_BASE_URL`.

### More than one vault in the container

The Local REST API plugin is per-vault, and every vault defaults to the same ports (`27123` HTTP / `27124` HTTPS). If two vaults have the plugin enabled, they fight over those ports: whichever loads first binds them, the other fails with `EADDRINUSE` and never starts a server at all. `/config/.config/obsidian/obsidian.json` lists every vault Obsidian currently has open.

This fails misleadingly. The port answers, so the network half looks healthy — but it is the _other_ vault answering, and its API key is a different one, so pasting the correct key for the vault you actually want gives:

```text
Token validation failed: invalid token
```

which reads like a bad key rather than a port collision. A `0.0.0.0` binding host makes it worse, not better: it claims the port on every interface and so beats a vault still sitting on `127.0.0.1`.

Pick one arrangement:

- **One vault exposed** — set the binding host on the vault you want, and disable the Local REST API plugin entirely in the other (remove it from `.obsidian/community-plugins.json`, or toggle it off in the GUI). Turning off just "Enable HTTP server" is not enough; `27124` still collides.
- **Both exposed** — give each vault its own port pair (e.g. `27125`/`27126` for the second) and set the binding host on both. OmniRoute still only talks to one of them, since it stores a single base URL.

Make these edits with Obsidian stopped (`docker stop obsidian`). A running Obsidian rewrites `data.json` from memory and will silently discard them.

### Vault path and WebDAV

The **vault path** setting is not part of the REST API integration, and leaving it empty costs nothing. Only `src/lib/obsidianSync.ts` reads it (the settings route just echoes it back for display); notes, context and the memory backend all go over the Local REST API, so the integration is complete with `vaultPath: null`.

A path like `/config/Desktop/Omniroute` is rejected because it is checked with `fs.existsSync()` inside the **omniroute** container, where it does not exist — it belongs to the `obsidian` container:

```text
Vault directory not found: /config/Desktop/Omniroute
```

`compose.yml` mounts the vault into the gateway at `/vault` (a `subpath` mount off the `obsidian-config` volume, scoped so the plugin's API key stays out of reach). Use `/vault` — never the other container's path.

Setting it has a side effect worth understanding first. The only route that writes it is `POST /api/settings/obsidian/webdav`, which in the same call switches on a WebDAV file server at `/api/v1/webdav` on the dashboard port and returns a generated Basic username/password. That handler runs ahead of Next.js and outside its authz pipeline (upstream tracks this as GHSA-7pq4-8pvv-rx7r), so those credentials are the only thing standing between the LAN and read/write access to the whole vault. `DELETE` on the same route turns it off again and clears the path.

## What's included

- **`compose.yml`** — the OmniRoute stack (`redis`, `omniroute`, `obsidian`), configured via env vars from `.env`.
- **`environment.yml` / `requirements.txt`** — conda environment (Python, pip, `gh`) with Python deps installed via pip.
- **`pyproject.toml`** — [tox](https://tox.wiki) environments for linting and formatting:
  - `lint` — `ruff check`
  - `format` — `ruff format` + `ruff check --fix` + `prettier --write` + `taplo fmt`
  - `txt-lint` — [textlint](https://textlint.github.io/) over `**/*.txt`
  - `prettier` — `prettier --check` over CSS/JS/HTML/JSON/YAML/Markdown
  - `toml-lint` — `taplo` format/lint check over TOML files
  - `duplicate-code` — [jscpd](https://github.com/kucherenko/jscpd) zero-tolerance duplicate-code scan (config in `.jscpd.json`)
  - `secret-detection` — [TruffleHog](https://github.com/trufflesecurity/trufflehog) scan of the full git history via Docker
  - `zizmor` — [zizmor](https://github.com/woodruffw/zizmor) static-analysis security scan of the GitHub Actions workflows themselves
  - `github` — the full read-only CI chain (`lint` + `txt-lint` + `prettier` + `toml-lint` + `duplicate-code`)
  - `all` — `format`, then `github`, then `secret-detection`
  - Also configures [git-cliff](https://git-cliff.org/) for generating changelogs/PR descriptions from Conventional Commits.
- **`package.json`** — `prettier` and `textlint` (+ plugins), installed on demand by the relevant tox envs.
- **`.github/workflows/`**
  - `tests.yml` — runs `tox -e github` on every push and PR.
  - `secrets.yml` — TruffleHog scan on every push and PR.
  - `zizmor.yml` — zizmor scan of `.github/workflows/` on every push to `main`.
  - `duplicate-code.yml` — jscpd scan.
  - `cascade-merge.yml` / `release.yml` — on push to `main`, bumps semver based on Conventional Commit prefixes (`feat` → minor, `fix`/other → patch, `!`/`BREAKING CHANGE` → major) and publishes a GitHub Release with an auto-generated changelog.
- **`CODEOWNERS`** — defaults review ownership to `@Self-Host-Server/code-owners`.
- **`.gitignore`** — editor/AI-assistant artifacts (`.vscode`, `.cursor`, `CLAUDE.md`, etc.), `.env`, `node_modules`.

## Development

1. Set up the environment (env name comes from `conda_name` in `.env`):

   ```bash
   make conda
   conda activate omniroute
   ```

2. Install `tox` and run the full check locally before pushing (requires Docker for `secret-detection`):

   ```bash
   pip install tox
   tox -e all      # format, then github (lint + txt-lint + prettier + toml-lint + duplicate-code), then secret-detection
   ```

   `tox -e github` runs everything except `secret-detection` — it's what CI's `tests.yml` runs, but it will **not** catch a leaked credential the way `tox -e all` (or CI's separate `secrets.yml`) does. `zizmor` is also standalone (own CI workflow, network access needed to install) — run it explicitly with `tox -e zizmor`.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the commit convention (Conventional Commits — it drives changelog generation and release versioning) and the local checks to run before opening a PR.
