# AX42 Monitoring Stack (ax42-monitor)

Standalone operational infrastructure: **Telegraf + InfluxDB + Grafana**
running on the AX42 Hetzner server via Coolify. Dashboard is embedded
inside the Stitchflow WMS under **Server → Health**, but this repo is
independent — you can run it without Stitchflow if you want plain Grafana
at `metrics.fournico.com`.

## Why this stack and not something in Supabase?

| Aspect | Custom Supabase table | This stack (chosen) |
|---|---|---|
| Supabase storage impact | ~9 MB/month | **zero** |
| Custom schema / RLS / Edge Function | Yes | No |
| Custom React charts | Yes | No (Grafana native) |
| Retention cleanup | pg_cron job | Native InfluxDB TTL |
| Alerting | Build from scratch | Grafana native (wired to ntfy) |
| Multi-panel / zoom / annotations | Build from scratch | Native |

See also `MedusaJS Logs` (Server → MedusaJS Logs page): that writes structured
application logs into Supabase via a Winston transport. This stack is for
*operational* metrics only — CPU, memory, disk, temperatures, Docker.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  AX42 server (Coolify)                                  │
│                                                          │
│  ┌─────────────┐    ┌──────────────┐   ┌─────────────┐ │
│  │  Telegraf   │ ─→ │  InfluxDB 2  │ ←─│   Grafana   │ │
│  │ (container) │    │  (container) │   │ (container) │ │
│  └─────────────┘    └──────────────┘   └──────┬──────┘ │
│    reads                 stores                │        │
│    /proc, /sys,          60-day TTL            │        │
│    /var/run/docker.sock                        │        │
│                                                │        │
│                      exposed via Coolify ──────┼────┐   │
└────────────────────────────────────────────────┼────┼───┘
                                                 │    │
                            metrics.fournico.com │    │
                                                 │    │
┌────────────────────────────────────────────────┼────┘
│  Stitchflow (order-picker-dashboard-ui)        │
│                                                │
│  /server/health                                │
│   ┌────────────────────────────────────────┐  │
│   │  <iframe src="metrics.fournico.com/    │←─┘
│   │     d/server-health?kiosk=tv" />       │
│   └────────────────────────────────────────┘
└─────────────────────────────────────────────────────────┘
```

## Files in this repo

```
ax42-monitor/
├── README.md                   # You are here
├── docker-compose.yml          # Full stack — deploy via Coolify
├── .env.example                # Env var template
├── telegraf/
│   └── telegraf.conf           # Scrape config (cpu, mem, disk, docker, sensors)
└── grafana/
    ├── dashboards/
    │   └── server-health.json  # Pre-built dashboard (auto-loaded)
    └── provisioning/
        ├── datasources/
        │   └── influxdb.yml    # Auto-registers InfluxDB as datasource
        ├── dashboards/
        │   └── dashboards.yml  # Dashboard provisioning config
        └── alerting/
            ├── contact-points.yml       # ntfy webhook contact point
            ├── notification-policies.yml # Route all alerts to ntfy
            └── alert-rules.yml          # CPU/mem/disk/temp/docker alerts
```

The Stitchflow WMS side (nav item + iframe page) lives separately in the
`order-picker-dashboard-ui` repo at:
- `src/pages/server/ServerHealthPage.tsx` — iframe wrapper
- `src/components/wms/config/navigationConfig.ts` — "Health" nav entry
- `src/App.tsx` — route

## One-time host setup (SSH into the AX42)

### 1. Create persistent data directories

```bash
sudo mkdir -p /data/monitoring/influxdb
sudo mkdir -p /data/monitoring/grafana
sudo mkdir -p /data/monitoring/telegraf
sudo mkdir -p /data/monitoring/ntfy/cache
sudo mkdir -p /data/monitoring/ntfy/lib

# Grafana runs as UID 472 inside the container
sudo chown -R 472:472 /data/monitoring/grafana
# ntfy runs as UID 1000
sudo chown -R 1000:1000 /data/monitoring/ntfy
```

Coolify restarts shouldn't wipe these, since they live outside any Docker
volume and are bind-mounted from the host.

### 2. Install lm-sensors (for Ryzen CPU temperature)

```bash
sudo apt update
sudo apt install -y lm-sensors
sudo sensors-detect --auto

# Verify — should show Tctl or k10temp readings
sensors
```

If `sensors-detect` finds modules that need kernel loading, reboot once.

### 3. Find the Docker group ID

Telegraf reads `/var/run/docker.sock` and needs to run as a user in the
`docker` group. Get the host GID:

```bash
getent group docker | cut -d: -f3
```

Save this value — you'll paste it into the `DOCKER_GID` env var in Coolify.

## Coolify deployment

### 1. Create a new Service in Coolify

- **Type**: Docker Compose
- **Source**: Paste the contents of `infra/monitoring/docker-compose.yml`

### 2. Set environment variables

In the Coolify env var UI, add:

```
# InfluxDB bootstrap (read once on first boot)
INFLUX_ADMIN_USERNAME=admin
INFLUX_ADMIN_PASSWORD=<generate a strong password>
INFLUX_ORG=fournico
INFLUX_BUCKET=server_health
INFLUX_RETENTION=60d
# Generate with:  openssl rand -hex 32
INFLUX_TOKEN=<64 hex chars>

# Grafana
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=<generate a strong password>
GF_SERVER_DOMAIN=metrics.fournico.com
GF_SERVER_ROOT_URL=https://metrics.fournico.com

# ntfy (self-hosted in this stack)
NTFY_BASE_URL=https://ntfy.fournico.com
NTFY_AUTH_DEFAULT_ACCESS=read-write
# Pick an unguessable topic name (not "alerts" or "test"):
NTFY_WEBHOOK_URL=https://ntfy.fournico.com/ax42-alerts-k7dh9s3f

# From step 3 of host setup
DOCKER_GID=999
```

**Important**: `INFLUX_TOKEN` is used as BOTH the InfluxDB admin token
(during first-boot bootstrap) AND the Telegraf write token. Using a single
value keeps the stack simple.

### 3. Configure the service domains

In Coolify, for the `grafana` service:
- **Domain**: `https://metrics.fournico.com`
- **Port**: `3000`

For the `ntfy` service:
- **Domain**: `https://ntfy.fournico.com`
- **Port**: `80`

Leave `influxdb` and `telegraf` with no domain — they only talk internally.

Enable TLS for both (Coolify handles Let's Encrypt automatically). Point
DNS A records for **both** `metrics.fournico.com` and `ntfy.fournico.com`
at the AX42 IP **before** starting the service so Let's Encrypt can issue
the certs.

**Cloudflare note**: if your domains are proxied by Cloudflare (orange
cloud ☁️), Let's Encrypt HTTP-01 challenges will fail. Either set both
subdomains to **DNS only** (grey cloud) OR use a Cloudflare Origin
Certificate. For admin dashboards the grey-cloud approach is simpler.

### 4. Start the service

On first boot:
1. **InfluxDB** self-configures from the `DOCKER_INFLUXDB_INIT_*` env vars →
   creates the admin user, org, bucket, and writes the admin token
2. **Telegraf** waits for the InfluxDB health check to pass, then starts
   scraping every 60 seconds
3. **Grafana** auto-provisions:
   - The InfluxDB datasource (`influxdb.yml`)
   - The Server Health dashboard (`server-health.json`)
   - The ntfy contact point (`contact-points.yml`)
   - The notification policy (`notification-policies.yml`)
   - The alert rules (`alert-rules.yml`)

### 5. First login to Grafana

1. Open `https://metrics.fournico.com`
2. Log in with `GF_SECURITY_ADMIN_USER` / `GF_SECURITY_ADMIN_PASSWORD`
3. (Optional) Change the admin password on first login
4. Navigate to **Dashboards → Server Health** → you should see data points
   for CPU, memory, temperature, Docker containers

### 6. Subscribe to ntfy on your Android

1. Install the **ntfy** app from the Play Store (if you don't have it yet)
2. Open the app → tap **+** → **Subscribe to topic**
3. Tap **Use another server** and enter: `https://ntfy.fournico.com`
4. Topic name: use the same unguessable suffix you put in `NTFY_WEBHOOK_URL`
   (e.g. `ax42-alerts-k7dh9s3f`)
5. Tap **Subscribe**

The topic will appear in your subscription list. Keep the ntfy app in the
background — Android will wake it up when messages arrive.

### 7. Test the ntfy alert from Grafana

1. Back in Grafana, navigate to **Alerting → Contact points**
2. Find `ntfy-android` → click the **Test** button (paper airplane icon)
3. Your phone should buzz within ~1 second with a notification like:
   ```
   [FIRING] AX42 CPU usage high
   ```

If it doesn't arrive, check:
- Grafana logs: look for errors POSTing to `NTFY_WEBHOOK_URL`
- ntfy logs: `docker logs <stack>-ntfy-1` — should show incoming POST
- Is the ntfy app subscribed to the right topic on the right server?

### (Optional) Lock down ntfy with auth

By default the ntfy server uses `NTFY_AUTH_DEFAULT_ACCESS=read-write`
which means anyone with a topic URL can publish or subscribe. Since you
picked an unguessable topic name, this is "security through obscurity"
but works fine for most cases.

To require login instead:
```bash
# Change the env var in Coolify:
NTFY_AUTH_DEFAULT_ACCESS=deny-all

# SSH into the AX42 and exec into the ntfy container:
docker exec -it <stack>-ntfy-1 sh

# Create an admin user who can publish AND subscribe
ntfy user add --role=admin grafana
# (enter password when prompted)

# Grant that user access to the topic
ntfy access grafana "ax42-alerts-k7dh9s3f" read-write
```

Then update the Grafana contact point URL to include basic auth:
```
NTFY_WEBHOOK_URL=https://grafana:YOUR_PASSWORD@ntfy.fournico.com/ax42-alerts-k7dh9s3f
```

And subscribe on Android with the same credentials (ntfy app supports
per-server basic auth).

## Wiring the dashboard into Stitchflow

1. Copy the dashboard URL (from your browser's address bar when viewing the
   Server Health dashboard). It will look like:
   ```
   https://metrics.fournico.com/d/server-health/server-health
   ```

2. Add these env vars to the Stitchflow Coolify deployment:
   ```
   VITE_GRAFANA_HEALTH_URL=https://metrics.fournico.com/d/server-health/server-health?kiosk=tv
   VITE_GRAFANA_LOGIN_URL=https://metrics.fournico.com/login   # optional
   ```

   **Important**: `VITE_*` env vars are read at **build time**, not runtime.
   After setting them, redeploy Stitchflow so Vite bakes them into the
   bundle.

3. Open Stitchflow as admin → **Server → Health** → the dashboard should
   render inline. If it shows a Grafana login page, click the
   **Login to Grafana** button in the header, log in in the new tab, then
   come back and refresh.

### Why isn't it a "public dashboard"?

Grafana's "public dashboard" feature creates a URL that anyone with the
link can view — no authentication. That's a leak risk: the URL ends up in
browser history, screenshots, and the DOM (visible via Inspect Element).
While the exposed data is "only" server metrics, we don't need anyone
outside Fournico seeing them.

Instead, we rely on Grafana's normal login + the browser session cookie.
Since `metrics.fournico.com` and Stitchflow both live under the
`fournico.com` parent domain, Grafana's session cookie is available to the
iframe. You log into Grafana once per browser session; after that, the
iframe just works.

## Alerting

All alerts route to the `ntfy-android` contact point which POSTs to your
self-hosted ntfy topic. Within ~1 second of an alert firing, your Android
device receives a push notification.

Rules configured in `alert-rules.yml`:

| Rule | Condition | For | Severity |
|---|---|---|---|
| CPU usage high | `cpu.usage_active > 85%` | 5m | critical |
| Memory usage high | `mem.used_percent > 85%` | 5m | critical |
| Disk almost full | any mount `disk.used_percent > 90%` | 1m | critical |
| Load high | `system.load1 > 18` (1.5× n_cpus for a 12-core Ryzen) | 10m | warning |
| CPU overheating | `sensors.temp_input > 85°C` | 2m | critical |
| Docker container crashed | running count dropped >2 vs 24h max | 2m | critical |

To tune thresholds, edit `alert-rules.yml` and Coolify will reload the
config on next Grafana restart.

## Adding more metrics or dashboards

### Add a Telegraf input

Edit `infra/monitoring/telegraf/telegraf.conf`, add a `[[inputs.XXX]]`
block, and restart the Telegraf container. Examples:

- `[[inputs.redis]]` — Coolify's Redis cache
- `[[inputs.postgresql_extensible]]` — Medusa's Postgres DB
- `[[inputs.http_response]]` — probe fournico.com for uptime

### Add a Grafana dashboard

Drop a JSON file into `infra/monitoring/grafana/dashboards/`. Grafana scans
that folder every 30 seconds and auto-imports any new files.

You can also import community dashboards by UID from
[grafana.com/dashboards](https://grafana.com/dashboards):
- **928** — Telegraf System Metrics
- **1150** — Docker Host & Container Overview
- **9628** — PostgreSQL Database

## Troubleshooting

### Grafana shows "No data"

- Check Telegraf logs: `docker logs <stack>-telegraf-1` — look for
  "Error writing to output" or "Unable to resolve"
- Verify the InfluxDB token matches between `INFLUX_TOKEN` and whatever
  was actually set in the bucket (bootstrap only runs once!)
- From inside the Telegraf container:
  `curl -H "Authorization: Token $INFLUX_TOKEN" http://influxdb:8086/health`
  should return `{"status":"pass"}`

### Iframe in Stitchflow shows Grafana login page

Normal when the admin's Grafana session has expired. Click
**Login to Grafana** → log in in the new tab → refresh the Stitchflow page.

### Iframe shows "refused to connect"

Grafana is refusing to be embedded. Check that these env vars are set:

```
GF_SECURITY_ALLOW_EMBEDDING=true
GF_SECURITY_CONTENT_SECURITY_POLICY=false
```

Both are already in `docker-compose.yml` — if you edited it, make sure
they didn't get removed.

### CPU temperature panel is empty

- SSH into the AX42 and run `sensors`. If it outputs nothing, `lm-sensors`
  wasn't installed or `sensors-detect` wasn't run.
- Verify `/sys` is mounted into the Telegraf container (check
  `docker inspect <stack>-telegraf-1 | grep '/sys'`)
- Check the Telegraf logs for `[inputs.sensors]` errors

### Persisting data across Coolify rebuilds

Data is bind-mounted from `/data/monitoring/{influxdb,grafana,telegraf}`
on the host. Coolify rebuilds re-create containers but do NOT touch
these directories. If you ever want to reset (e.g., wipe metrics
history), `rm -rf /data/monitoring/influxdb/*` and restart the stack.

## Out of scope (future)

- **Medusa application metrics** via OpenTelemetry (Telegraf has an OTel
  input plugin). Would give us per-endpoint request rate, latency, error
  rate for the Medusa backend.
- **Grafana Loki** for log streaming. We already have `server_logs` in
  Supabase + the MedusaJS Logs page in Stitchflow, so probably not needed.
- **Grafana Image Renderer** plugin for PNG exports (for email reports,
  automated screenshots to Slack, etc.).
