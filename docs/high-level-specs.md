# foresight-web — Architecture & Deployment Spec

A React frontend + FastAPI backend, deployed as Docker containers on the
self-hosted mini PC (`foresight`), with branch-driven CI/CD.

- **`master` branch** → production, served publicly at `https://4sight.lol` (port `8080`)
- **`development` branch** → test, reachable over Tailscale at `foresight:8090`

**Host hardware:** BOSGAME P3 — AMD Ryzen 5 7640HS (6C/12T, Zen 4, up to 5.0 GHz),
32GB DDR5-4800 (dual channel), 1TB NVMe PCIe 4.0, dual 2.5G LAN. Plenty of
headroom to run Postgres + the foresight pipeline + both web stacks (test & prod)
concurrently.

---

## 1. System context — shared core, coordinated by Redis

Three pieces, sharing a common library and a Redis-backed rate limiter:

```
┌─────────────────────┐        ┌──────────────────────┐
│  foresight pipeline  │        │  foresight-web        │
│  (long-running jobs) │        │  backend (FastAPI)    │
└──────────┬───────────┘        └───────────┬──────────┘
           │  both import                    │
           │       foresight-core            │
           │  (RiotAPIClient, Region,        │
           │   DB write helpers,             │
           │   rate-limit client)            │
           ▼                                 ▼
        ┌──────── Redis ────────┐   shared rate-limit budget
        │  call-quota counters  │◄──── both draw from one bucket
        └───────────────────────┘
           │                                 │
           ▼                                 ▼
        ┌──────────────── Postgres ────────────────┐
        │  public schema (prod)  /  test schema     │
        └───────────────────────────────────────────┘
```

### The model (Option A — web calls Riot directly)

The web backend **imports `foresight-core` and calls Riot directly** when it
needs a synchronous result. Example: a user triggers an account update → the
backend calls `foresight_core.update_user(...)`, which hits Riot, writes to
Postgres, **and returns the fresh result in the same request** so the UI updates
immediately. The pipeline uses the *same* `foresight-core` functions for its
long-running batch jobs.

### Why this is safe: the Redis rate limiter

The risk of two callers (web + pipeline) hitting Riot independently is that they
compete for the single API-key quota. **Redis solves this**: the rate-limit
state lives in Redis, not inside either process, so both draw from **one shared
budget**. Whoever calls — web request or pipeline job — decrements the same
counter. This is the key that makes direct synchronous calls from the web app
safe alongside the pipeline.

### `foresight-core` (shared package)

Split into its **own repo** (`foresight-core`), holding:
- `RiotAPIClient`, `Region`, and related API-model code
- DB write/read helpers
- the Redis rate-limit client

Both the pipeline and the web backend declare it as a versioned dependency
(`pip install git+...@vX` or a pinned tag). One source of truth, no copy-paste
drift. The web backend's Dockerfile installs it at build time like any dep.

### Synchronous vs. job-queue (deferred)

Default is **synchronous direct calls** — simplest, gives immediate results.
A **job-queue path is optional and can be added later** without changing
`foresight-core`: the queue would just be another caller of the *same* functions
(web writes a job row → pipeline executes it). Add it only when a use case needs
it (e.g. a slow bulk operation you don't want blocking a request). Until then,
synchronous covers it. *(Decision left open — neither path is foreclosed.)*

---

## 2. Repository layout (monorepo)

One repo, two cleanly separated apps:

```
foresight-web/
├── frontend/                 # React (Vite) — its own deps + Dockerfile
│   ├── Dockerfile
│   ├── package.json
│   └── src/
├── backend/                  # FastAPI — its own deps + Dockerfile
│   ├── Dockerfile
│   ├── pyproject.toml
│   └── app/
│       └── main.py
├── Caddyfile                 # reverse proxy config
├── docker-compose.prod.yml   # production stack
├── docker-compose.test.yml   # test stack
├── .env                      # single env on the box (NOT committed)
└── .github/
    └── workflows/
        ├── ci.yml            # PRs → cloud runner → build + test
        └── deploy.yml        # merges → self-hosted runner → deploy
```

Frontend and backend are independent apps (own Dockerfiles, own deps, talk over
HTTP). They live in one repo so a single PR can change both sides atomically and
one pipeline covers the whole site. The backend additionally depends on the
external **`foresight-core`** package (separate repo) for Riot access + the
Redis rate limiter.

---

## 3. How frontend and backend connect (why the Caddy proxy exists)

### The problem it solves

Your React app runs **in the visitor's browser**, not on the box — the box just
sends the files. So API calls originate from the *visitor's browser* and must
reach your backend. The frontend (static files) and backend (FastAPI) are **two
separate programs on two separate ports**. Something has to sit at the front,
receive every request to `4sight.lol`, and route it to the right program.

Without a front-door router, the browser would have to address each program by
its own port — and **a different port is a different origin**. The moment the
browser calls the backend on a different origin, the **same-origin policy** kicks
in and requires **CORS** configuration (permission headers + preflight
round-trips) to allow it. More config, a second exposed port, and extra latency.

### The fix

**Caddy** is a small reverse proxy (a traffic director). It's the single program
listening at the public port. It routes by path:

```
4sight.lol ──tunnel──> Caddy (:8080)
                         │  /api/*  → backend  (FastAPI, internal)
                         │  /       → frontend (static files, internal)
```

- `4sight.lol/`      → React app
- `4sight.lol/api/*` → FastAPI

The browser only ever sees **one origin** (`4sight.lol`), so the same-origin
policy is satisfied and **CORS never enters the picture**. The frontend and
backend ports stay internal to the Docker network — only Caddy's port faces the
tunnel.

### Cost / footprint

Caddy is free (open source, no tiers, no usage limits). The container is ~50MB on
disk and idles at ~10-15MB RAM. Its proxy→app hops are localhost (microseconds) —
*less* latency than the CORS preflight round-trips you'd otherwise incur. Config
is ~6 lines (below). Only real downside: one more service in the chain (auto-
restarts on failure via Docker, so low risk).

---

## 4. The two environments

Both run simultaneously on the box, fully isolated:

| | Production | Test |
|---|---|---|
| Trigger | merge to `master` | merge to `development` |
| Caddy port (on box) | `8080` | `8090` |
| Reachable via | Cloudflare tunnel → `4sight.lol` | Tailscale only (`foresight:8090`) |
| Postgres schema | `public` | `test` |
| Compose project | `foresight-prod` | `foresight-test` |
| `APP_ENV` value | `prod` | `test` |

Isolation points:
- **Different compose project names** (`-p foresight-prod` / `-p foresight-test`)
  so containers, networks, and volumes never collide.
- **Same Postgres instance, different schema** — reuses the `public` / `test`
  split already configured. Friends' experimental code (test) hits the `test`
  schema, never prod data.
- **Prod** Caddy binds `127.0.0.1:8080` → reachable only by the Cloudflare tunnel
  (which already points here — no tunnel change needed).
- **Test** Caddy binds the Tailscale IP `100.123.83.105:8090` → only tailnet
  members reach it.

---

## 5. Single env + APP_ENV selector

One `.env` file on the box holds all keys for both environments, e.g.
`API_KEY_PROD` / `API_KEY_TEST`, plus shared DB connection info. A single
`APP_ENV` value (`prod` or `test`), set per-deploy, tells the backend which keys
and which schema to use.

- `APP_ENV=prod` → reads `*_PROD` keys, `DB_SCHEMA=public`
- `APP_ENV=test` → reads `*_TEST` keys, `DB_SCHEMA=test`

(Env file *contents* are out of scope here — detailed later. Note: a single file
means both environments' secrets live together; acceptable for solo+friends,
revisit if stricter isolation is ever needed.)

The `.env` lives at `/home/lucian/foresight-web/.env`, **never committed**.

---

## 6. CI/CD split (the key design point)

**CI (testing) and CD (deploying) run in different places — your gated-merge
workflow is fully preserved:**

```
PR into development ─> GitHub CLOUD runner: build + lint + test ─> ✅ on PR ─> merge
                                                                                 │
merge to development ─> SELF-HOSTED runner (on box): deploy TEST stack (:8090)

PR into master ──────> GitHub CLOUD runner: build + lint + test ─> ✅ on PR ─> merge
                                                                                 │
merge to master ─────> SELF-HOSTED runner (on box): deploy PROD stack (:8080)
```

- **CI on cloud runners** — identical to existing foresight CI. PR checks you
  watch go green before merging. Needs zero access to the box.
- **CD on a self-hosted runner installed on the box** — runs *after* merge, does
  the actual `docker compose up`. Already on the box (behind CGNAT), polls GitHub
  outbound, so no inbound access or SSH-from-cloud needed.

Branch protection (Rulesets, like foresight uses): require the CI check to pass
before a PR can merge into `development` or `master`. That enforces
"wait for green before merge."

**Security note:** the self-hosted runner executes repo workflow code on the box.
Keep the repo **private** and branch protection on — anyone who can merge can run
commands on the box via the deploy workflow.

---

## 7. Workflow files (skeletons)

### `.github/workflows/ci.yml` — PRs, cloud runner

```yaml
name: CI
on:
  pull_request:
    branches: [development, master]

jobs:
  backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.14" }
      - run: pip install -e backend[dev]
      - run: ruff check backend
      - run: pytest backend

  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "22" }
      - run: npm ci --prefix frontend
      - run: npm run lint --prefix frontend
      - run: npm run build --prefix frontend
      - run: npm test --prefix frontend
```

### `.github/workflows/deploy.yml` — merges, self-hosted runner

```yaml
name: Deploy
on:
  push:
    branches: [development, master]

jobs:
  deploy:
    runs-on: [self-hosted, foresight]
    steps:
      - uses: actions/checkout@v4

      - name: Pick environment
        id: env
        run: |
          if [ "${{ github.ref_name }}" = "master" ]; then
            echo "compose=docker-compose.prod.yml" >> "$GITHUB_OUTPUT"
            echo "project=foresight-prod"          >> "$GITHUB_OUTPUT"
            echo "appenv=prod"                      >> "$GITHUB_OUTPUT"
          else
            echo "compose=docker-compose.test.yml" >> "$GITHUB_OUTPUT"
            echo "project=foresight-test"          >> "$GITHUB_OUTPUT"
            echo "appenv=test"                      >> "$GITHUB_OUTPUT"
          fi

      - name: Build & deploy
        run: |
          APP_ENV=${{ steps.env.outputs.appenv }} \
          docker compose \
            -p ${{ steps.env.outputs.project }} \
            --env-file .env \
            -f ${{ steps.env.outputs.compose }} \
            up -d --build

      - name: Prune old images
        run: docker image prune -f
```

---

## 8. Container Dockerfiles (skeletons)

### `backend/Dockerfile`

```dockerfile
FROM python:3.14-slim
WORKDIR /app
COPY pyproject.toml .
# foresight-core is a dependency in pyproject (pinned git tag or private index).
# If it's a PRIVATE repo, this RUN needs build-time auth — see
# "Installing the private foresight-core" below; do NOT use a plain git URL here.
RUN pip install --no-cache-dir -e .
COPY app/ app/
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### `frontend/Dockerfile` (multi-stage: build, then serve static)

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json .
RUN npm ci
COPY ../../../Downloads .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

(The frontend uses nginx **only to serve its own static files internally** — it
does *not* do API routing. Caddy is the front-door proxy that routes between
frontend and backend.)

### Installing the private `foresight-core` at build time

If `foresight-core` is a **private** repo, the Docker build can't interactively
authenticate — `pip install git+https://github.com/...` will fail on a private
URL with no credentials. The build needs auth supplied non-interactively. Three
approaches, cleanest first:

**A) BuildKit SSH mount + deploy key (recommended for git installs).**
Add a read-only **deploy key** to the `foresight-core` repo (GitHub → repo →
Settings → Deploy keys), keep its private key on the box, and mount it into the
build only for the install step so it never persists in an image layer:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.14-slim
RUN apt-get update && apt-get install -y --no-install-recommends git openssh-client \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY pyproject.toml .
RUN mkdir -p -m 0700 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts
# SSH agent socket is mounted ONLY for this RUN — not baked into the image
RUN --mount=type=ssh pip install --no-cache-dir -e .
COPY app/ app/
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```
`pyproject.toml` references it over SSH:
`foresight-core @ git+ssh://git@github.com/Git-Lukyen/foresight-core.git@v0.1.0`

In the deploy workflow, the self-hosted runner builds with the box's SSH agent:
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/foresight_core_deploy_key
docker compose ... build  # BuildKit forwards the agent via --mount=type=ssh
```
The key is mounted transiently, **never written into a layer** — the safe pattern.

**B) BuildKit secret mount + token (if you prefer HTTPS + a PAT).**
Use a fine-grained GitHub token with read-only access to `foresight-core`, mounted
as a build secret:
```dockerfile
RUN --mount=type=secret,id=gh_token \
    pip install --no-cache-dir \
    "foresight-core @ git+https://$(cat /run/secrets/gh_token)@github.com/Git-Lukyen/foresight-core.git@v0.1.0"
```
Build with `docker build --secret id=gh_token,src=token.txt ...`. Like the SSH
mount, the secret isn't persisted in the image. Avoid the naive
`ARG GH_TOKEN` / `ENV` approach — those **do** leak into image history.

**C) Private package index (cleanest long-term).**
Publish `foresight-core` to **GitHub Packages** (or a private PyPI). The Dockerfile
then does a normal `pip install foresight-core==0.1.0` with index credentials
supplied as a build secret. More upfront setup (a publish step in
`foresight-core`'s own CI), but the web build stays a plain `pip install` with no
git in the image, and versioning is explicit. Worth migrating to once
`foresight-core` stabilizes.

**Key rule across all three:** the credential must be a *transient build mount*
(`--mount=type=ssh` or `--mount=type=secret`), never an `ARG`/`ENV`/`COPY`, so it
doesn't end up readable in the final image's layer history.

---

## 9. Compose files (skeletons)

### `docker-compose.prod.yml`

```yaml
services:
  proxy:
    image: caddy:alpine
    ports:
      - "127.0.0.1:8080:80"      # tunnel reaches this
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
    depends_on: [frontend, backend]

  frontend:
    build: ./frontend

  backend:
    build: ./backend
    environment:
      APP_ENV: ${APP_ENV}
      DB_HOST: 100.123.83.105     # host Tailscale IP (or host.docker.internal)
      DB_PORT: "7012"
      DB_NAME: foresight
      DB_SCHEMA: public
      REDIS_URL: redis://redis:6379/0
    extra_hosts:
      - "host.docker.internal:host-gateway"
    depends_on: [redis]

  redis:
    image: redis:alpine
    # rate-limit state shared with the pipeline; persists quota counters
    volumes:
      - redis-data:/data

volumes:
  redis-data:
```

### `docker-compose.test.yml`

Identical, except:
```yaml
  proxy:
    ports:
      - "100.123.83.105:8090:80"  # bind to Tailscale IP → tailnet-only
  backend:
    environment:
      DB_SCHEMA: test
```

### `Caddyfile`

```
:80 {
    handle /api/* {
        reverse_proxy backend:8000
    }
    handle {
        reverse_proxy frontend:80
    }
}
```

---

## 10. Redis topology (important)

For the shared rate limiter to actually work, **the pipeline and the web backend
must point at the SAME Redis instance** — otherwise each has its own counters and
they don't coordinate, defeating the purpose.

Recommended: run **one Redis on the host** (or one shared container), reachable by
all callers, rather than a Redis baked into each compose stack. Options:

- **Host-level Redis** (`apt install redis-server`), bound to localhost +
  Tailscale IP, with all stacks using `REDIS_URL=redis://100.123.83.105:6379`.
  Single shared instance, survives stack rebuilds. **Preferred.**
- Or a single dedicated `redis` container that *both* the pipeline and web stacks
  connect to (not one-per-stack).

**Prod vs test sharing:** decide whether prod and test web stacks share the same
rate-limit budget (they hit the same Riot key, so usually **yes** — use the same
Redis, maybe different logical DB numbers `/0` prod, `/1` test). The compose
snippets above show a per-stack `redis` for brevity; collapse to one shared
instance per the above before relying on coordinated limits.

---

## 11. Postgres access from containers

Postgres runs on the **host** (not a container), so containers reach it via the
host's Tailscale IP `100.123.83.105:7012`, which the existing
`pg_hba.conf` rule (`100.64.0.0/10`) already permits. Alternatively use
`host.docker.internal` with the `host-gateway` mapping shown above.

The web backend writes to Riot-fed tables via `foresight-core` (same helpers the
pipeline uses), so it needs write access to the relevant schema — give it a
dedicated Postgres role scoped appropriately (prod→`public`, test→`test`),
narrower than a full superuser. Define its exact grants when the table layout is
known.

---

## 12. Cloudflare tunnel

Prod runs on `8080`, which the tunnel **already points at** — no change needed.
Once the prod stack is up, `4sight.lol` serves it automatically.

The **test** stack is intentionally *not* added to the tunnel — it stays
Tailscale-only.

---

## 13. One-time setup checklist

- [ ] Create `foresight-core` repo (Riot client + Redis rate limiter + DB helpers)
- [ ] Add a read-only **deploy key** (or fine-grained token) for `foresight-core` build auth
- [ ] Create private `foresight-web` repo, push `frontend/` + `backend/` skeletons
- [ ] Install Docker + compose plugin on the box
- [ ] Install Redis on the box (shared by pipeline + web stacks)
- [ ] Install GitHub Actions self-hosted runner on the box, label it `foresight`
- [ ] Create `/home/lucian/foresight-web/.env` (keys + DB + REDIS_URL)
- [ ] Add `Caddyfile`, compose files, Dockerfiles
- [ ] Create a dedicated web Postgres role (scoped to schema)
- [ ] Point pipeline + web at the SAME Redis for shared rate limiting
- [ ] Set branch protection (Rulesets) requiring CI on `development` + `master`
- [ ] First merge to `development` → confirm `foresight:8090` over Tailscale
- [ ] First merge to `master` → confirm `https://4sight.lol`

---

## 14. Daily workflow (what it feels like)

1. Branch off `development`, build a feature (frontend + backend together).
2. Open PR into `development` → cloud CI runs → wait for ✅.
3. Merge → box auto-deploys test → check at `foresight:8090` (you + friends).
4. When test looks good, PR `development` → `master` → CI ✅ → merge.
5. Box auto-deploys prod → live at `https://4sight.lol`.

Same gated-merge rhythm as foresight, now with automatic deploys on both ends.
