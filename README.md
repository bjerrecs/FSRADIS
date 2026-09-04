# FSRADIS

FSRADIS is a standalone, local-only prototype project that can later be integrated into FlightStrips.

It contains a React + TypeScript frontend that pulls live flight plan data directly from the [VATSIM data feed](https://data.vatsim.net/v3/vatsim-data.json).

## Why this is safe for FlightStrips

- It lives in its own folder: `FSRADIS/`
- It has separate dependencies and scripts
- No imports or runtime coupling with existing FlightStrips services

## Project structure

- `frontend/` - Vite + React + TypeScript UI

## Quick start

```bash
cd frontend
npm install
npm run dev
```

Open the frontend URL shown by Vite (usually `http://localhost:5173`).

## VATSIM data

- Flight plans for `EKYT` arrivals/departures are pulled from `https://data.vatsim.net/v3/vatsim-data.json`.
- The feed is polled every 15 seconds from the browser.
- If the feed is temporarily unavailable, the previously loaded strips are kept until the next successful poll.

## Docker

The app is a fully client-side SPA — it talks to the VATSIM feeds straight from
the browser, so the container is just a static build served by nginx. There is
no backend, no database and no configuration or secrets to supply.

The image is built in two stages (`node:20-alpine` builds the bundle,
`nginx:1.27-alpine` serves it) and lands at roughly 74 MB.

### Build and run locally

```bash
docker compose up -d --build
```

Then open `http://localhost:8080`.

Or without compose:

```bash
docker build -t fsradis:latest .
docker run -d --name fsradis -p 8080:80 fsradis:latest
```

### What nginx does

- Serves the SPA with a history fallback, so deep links resolve to `index.html`.
- Caches hashed `/assets/*` for a year; never caches `index.html` or
  `standassignment.txt`.
- Serves `/healthz` for the container healthcheck and any external probe.
- Sends gzip plus `X-Content-Type-Options`, `X-Frame-Options` and
  `Referrer-Policy`.

### Updating stand assignments without a rebuild

`standassignment.txt` is read at runtime, so it can be bind-mounted over. Both
compose files carry a commented-out `volumes:` block for this — uncomment it and
point it at your own copy.

## Deploying to Portainer

Two options, depending on whether you want Portainer to build the image or pull
a prebuilt one.

### Option A — pull the prebuilt image (recommended)

`.github/workflows/docker-publish.yml` publishes to GitHub Container Registry on
every push to `main` and on `v*` tags. In Portainer:

1. **Stacks → Add stack → Web editor**.
2. Paste the contents of `docker-compose.prod.yml` — it already points at
   `ghcr.io/bjerrecs/fsradis:latest`.
3. **Deploy the stack**.

The GHCR package inherits the repository's public visibility on first publish,
so no registry credentials are needed. If you later make the repo private, add a
credential under **Registries** (a GitHub PAT with `read:packages`).

To update later, hit **Pull and redeploy** on the stack.

### Option B — build from this repository

1. **Stacks → Add stack → Repository**.
2. Repository URL: this repo. Compose path: `docker-compose.yml`.
3. **Deploy the stack** — Portainer clones and builds the image on your host.

## Behind Nginx Proxy Manager

Deploy with `docker-compose.npm.yml`, which joins the proxy's Docker network and
publishes no host port. Then in NPM, **Details**:

| Field | Value |
| --- | --- |
| Domain Names | your hostname |
| Scheme | `http` |
| Forward Hostname / IP | `fsradis` |
| Forward Port | `80` |

Two things trip this up:

- **The forward port is 80, not 8080.** `8080` is a *host* published port; it
  does not exist on the container's own address. Pointing the proxy at
  `<container-ip>:8080` gives a connection refused, and NPM reports 502.
- **Use the container name, not its IP.** Container IPs are reassigned on
  redeploy, so a hard-coded `172.x.x.x` breaks the next time you pull and
  redeploy. The name resolves only if both containers share a network, which is
  what the compose file arranges.

On the **SSL** tab, enable **Force SSL** so the site is not served over plain
HTTP. **Cache Assets** should stay off -- this container already sends correct
cache headers, and NPM's caching would also cache `index.html`, pinning clients
to a stale asset manifest. **Websockets Support** is harmless but unused.

### Port

The build and prod compose files publish `8080:80`. Change the left-hand number if 8080 is
taken, or drop `ports:` entirely and attach it to your reverse proxy network.
