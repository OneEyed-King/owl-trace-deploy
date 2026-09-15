# owl-trace

Self-hosted observability for a Docker Compose stack — service health, request
tracing (via eBPF, zero code changes), and logs, in one dashboard, running
entirely on your own machine.

This repo is just the install file and instructions. It exists so you can spin
up owl-trace with a single command, without cloning the actual source.

Licensed under the [Elastic License 2.0](./LICENSE) — you can self-host and
use owl-trace freely, including for internal production use; you just can't
offer it to others as a hosted/managed service. See `LICENSE` for the full
terms.

## Requirements

- Linux host (Beyla's eBPF tracing needs a Linux kernel — this won't work on
  macOS/Windows directly, only inside a Linux VM)
- Docker Engine with the Compose plugin (`docker compose version` should work)
- Enough free RAM for ClickHouse — 1–2 GB is comfortable for a demo-sized
  stack

## Turning it on

```bash
curl -fsS https://raw.githubusercontent.com/OneEyed-King/owl-trace-deploy/main/docker-compose.yaml \
  | docker compose -f - up -d
```

That pulls four prebuilt images (`engine`, `web`, `beyla`, `clickhouse`) and
starts them. First run takes a minute or two while images download.

Once it's up, open **http://localhost:8973** (or the port you chose below).
On a remote server, see [Accessing it remotely](#accessing-it-remotely)
before doing anything with firewalls.

### Port options

Nothing else on your machine should be using these already. If it is, set
the environment variables before the `curl`:

```bash
OWLTRACE_UI_PORT=9000 OWLTRACE_OTLP_PORT=4319 \
  curl -fsS https://raw.githubusercontent.com/OneEyed-King/owl-trace-deploy/main/docker-compose.yaml \
  | docker compose -f - up -d
```

| Variable | Default | What it's for |
|---|---|---|
| `OWLTRACE_UI_PORT` | `8973` | The dashboard itself |
| `OWLTRACE_OTLP_PORT` | `4318` | Where any service you fully instrument sends trace data |
| `OWL_ENGINE_API_KEY` | a fixed dev default | Shared secret between the two internal `web`/`engine` containers — only matters if you later expose the engine beyond this one Docker network |

ClickHouse's own port is **not published** by default (see
[Data & privacy](#data--privacy) for why) — if you want to point your own
Grafana at it, see the note at the bottom of `docker-compose.yaml`.

## How it works

- **Beyla** runs as a privileged container using eBPF to watch network traffic
  on the host — it sees HTTP/gRPC requests between your other containers
  without any code changes, agent installs, or restarts to those containers.
- **engine** receives that trace data (and separately polls the Docker socket
  for container health/CPU/memory), stores it in **ClickHouse**, and serves a
  small internal API.
- **web** serves the dashboard UI and proxies API calls to `engine`.
- Everything talks to everything else over one internal Docker network
  (`owl-trace`) that this compose file creates — nothing here talks to any
  other stack you have running unless you explicitly opt a service into
  deeper tracing.

## Data & privacy

Everything owl-trace collects — traces, logs, container metrics — stays in
the ClickHouse container running on your own host. Nothing is sent to us
(Observingowl Studios) or any third party by default.

Two honest exceptions, both entirely under your control:

- **Alert notifications** — if you configure a Telegram bot or a webhook URL
  in the alerts settings, owl-trace will send messages *there* when something
  trips an alert. That's a destination you chose, not a hidden default.
- **Optional instrumentation agents** — if you opt a specific service into
  full OpenTelemetry auto-instrumentation (a separate, deeper tracing mode
  than Beyla's default eBPF capture), owl-trace downloads the relevant public
  OpenTelemetry agent jar/package from GitHub's own release URLs the first
  time you turn that on. That's a one-time *download*, not anything being
  sent out.

Outside of those two cases, nothing phones home, and there's no telemetry or
analytics baked into the tool itself.

## Using it

1. Open the dashboard and you'll see every container Docker knows about,
   listed as "discovered."
2. Click a service to start "watching" it — this turns on health/CPU/memory
   polling and, if Beyla already sees HTTP traffic to it, live traces show up
   automatically with no extra setup.
3. Each watched service has tabs for recent traces, live/clustered/historical
   logs, and metrics history. Logs are read live from Docker by default; the
   "History" tab lets you additionally opt a service into ClickHouse-backed
   log storage if you want searchable, longer-retained logs (this uses more
   disk — the UI explains the tradeoff before you turn it on).
4. Retention for both traces and stored log patterns is configurable from the
   dashboard (a day-count preset, capped sensibly, with a live estimate of how
   much disk it'll use).

Services that show "no traces yet" are either genuinely idle (no traffic to
capture) or don't speak an HTTP/gRPC-shaped protocol Beyla can see as a trace
at all (e.g. Kafka, InfluxDB, Telegraf) — that's expected, not a
misconfiguration.

## Accessing it remotely

Running owl-trace on a VPS/cloud box instead of your laptop? Docker
publishing a port (what this compose file does) is separate from your
provider's firewall — both have to allow it.

**Just testing privately, don't want to touch firewall rules at all:**

```bash
ssh -L 8973:localhost:8973 you@your-server
```

Then open `http://localhost:8973` in your own browser as if it were local.
Nothing on the server's firewall changes; the tunnel does the work.

**Want to show someone else the dashboard directly:**
Open only `OWLTRACE_UI_PORT` (8973 by default) in your provider's firewall /
security group. Do **not** open ClickHouse's port for this — it has no
authentication, and the compose file doesn't publish it by default on
purpose. This is genuinely a real, unencrypted-over-plain-HTTP port at this
stage — fine for a quick demo to someone you trust the link with, not
something to leave open long-term or put a real password-protected workflow
behind. A domain + TLS setup (reverse proxy) is a natural next step but isn't
covered by this quick-start.

## Removing it

```bash
docker compose -p owl-trace down -v --rmi all
```

- `-v` removes the named volumes — this is what actually frees the
  ClickHouse data and owl-trace's own state, not just `down` alone.
- `--rmi all` removes all four images referenced by the compose file.

If Beyla's container refuses to stop (`container ... PID ... is zombie and
can not be killed` — a known quirk of eBPF containers without their own
init), force-remove it first, then re-run the command above:

```bash
docker rm -f owl-trace-beyla
docker compose -p owl-trace down -v --rmi all
```

Verify nothing owl-trace-related is left:

```bash
docker ps -a --filter "label=com.docker.compose.project=owl-trace"
docker volume ls --filter "label=com.docker.compose.project=owl-trace"
docker images | grep -E "owl-trace|clickhouse"
```

All three should come back empty.
