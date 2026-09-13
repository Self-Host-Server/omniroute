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

## What's included

- **`compose.yml`** — the OmniRoute stack (`redis`, `omniroute`), configured via env vars from `.env`.
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
