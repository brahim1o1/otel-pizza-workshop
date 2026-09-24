# OpenTelemetry Pizza Workshop

A small pizza-ordering app built from three Node services and a web frontend.
The three Node services report traces, metrics and logs to Dash0 through
OpenTelemetry's zero-code instrumentation — see
[Sending telemetry to Dash0](#sending-telemetry-to-dash0) below.

## Prerequisites

- **Docker Desktop** — [download here](https://www.docker.com/products/docker-desktop)
- **Node.js** v18 or higher — [download here](https://nodejs.org/) (only needed
  once you start changing the services)
- Ports `3000`, `3001`, `3002` and `8080` free

## Get the code

**Fork first, then clone your fork.** Later stages of the workshop open pull
requests against your repository, so you need to own the remote.

1. Open <https://github.com/dash0-community/otel-pizza-workshop> and click **Fork**.
   Keep the default name.

2. Clone your fork and keep a link to this repository:

```bash
# replace YOUR-USERNAME with your GitHub username
git clone https://github.com/YOUR-USERNAME/otel-pizza-workshop.git
cd otel-pizza-workshop

git remote add upstream https://github.com/dash0-community/otel-pizza-workshop.git
git remote -v
```

## Sending telemetry to Dash0

The services read their Dash0 credentials from `pizza-app/.env`. Create it
before the first run:

```bash
cd pizza-app
cp .env.template .env
```

Then edit `.env` and set:

- `DASH0_AUTH_TOKEN` — from **Settings → Auth Tokens** in Dash0
- `DASH0_ENDPOINT` — the ingress endpoint for your region, listed in the template

`.env` is gitignored, so your token stays out of the repository. Compose
refuses to start without both values rather than running blind.

No tracing code was added to the services. Each one starts with
`node --require @opentelemetry/auto-instrumentations-node/register`, which
instruments Express, the HTTP client and Pino before the app loads, and every
other setting is an `OTEL_*` environment variable in `docker-compose.yml`.

What arrives in Dash0:

| Signal | Comes from |
|---|---|
| Traces | One trace per order, spanning all three services. Trace context rides the `traceparent` header between them, so a failing order shows the whole chain. |
| Logs | Existing Pino `logger.*` calls, each stamped with the trace and span it happened in |
| Metrics | HTTP server and client request durations, Node event-loop delay, V8 heap |

## Run it

```bash
docker compose up
```

The first build takes a few minutes. When all four containers are up, open
<http://localhost:8080> and order a pizza.

Stop with `Ctrl+C`, or:

```bash
docker compose down            # stop
docker compose down -v         # and volumes
docker compose down --rmi all  # and images
```

## What's running

| Service | Port | Does |
|---|---|---|
| Frontend | 8080 | Order form |
| Order Service | 3000 | Takes the order, calls the other two |
| Kitchen Service | 3001 | Checks availability, cooks |
| Delivery Service | 3002 | Assigns a driver |

Logs from all four are interleaved in the terminal you ran `docker compose up`
in. For one service on its own:

```bash
docker compose logs -f order-service
```

## Failure modes you can switch on

```bash
SLOW_KITCHEN=true docker compose up   # the oven takes ~5s per pizza
NO_DRIVERS=true docker compose up     # nobody is available to deliver
```

## Troubleshooting

**Containers won't start** — check Docker Desktop is running, then
`docker compose down && docker compose up --build`.

**Port already in use** — something else holds 3000, 3001, 3002 or 8080.
`lsof -i :3000` will name it.

**Changed a file and nothing happened** — the services are baked into images.
Rebuild: `docker compose up -d --build`.

**Compose won't start, complains about `DASH0_ENDPOINT` or `DASH0_AUTH_TOKEN`** —
`pizza-app/.env` is missing or incomplete. See
[Sending telemetry to Dash0](#sending-telemetry-to-dash0).

**Nothing shows up in Dash0** — set `OTEL_LOG_LEVEL=debug` in `pizza-app/.env`
and restart; the SDK then logs what it is exporting and any rejection it gets
back. Check the token's permissions allow ingesting, and that `DASH0_ENDPOINT`
matches your region.

**Build fails** — `docker compose build --no-cache`, and check the JSON in any
`package.json` you edited.

## Credits

The pizza app and the original workshop are the work of
[Julia Morgado](https://github.com/juliafmorgado/otel-pizza-workshop).

## License

MIT License — feel free to use this for learning!
