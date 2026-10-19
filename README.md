# Data Pipeline Template

Dockerized Node-RED -> InfluxDB -> Grafana stack for pulling machine/process
data off the plant floor and into dashboards clients can actually use.

This is a **template repo** — clone it per client (`git clone` into a new
private repo, or use it as a GitHub template), customize the input node and
the business logic in `node-red/data/flows.json`, and ship it. See
`docs/ARCHITECTURE.md` for how the pipeline is structured and
`docs/ARCHITECTURE.md#going-from-simulator-to-a-real-input` for wiring up a
client's actual equipment.

## Stack

| Component | Role |
|---|---|
| [Node-RED](https://nodered.org/) | Ingestion, routing, and transformation |
| [InfluxDB 2.x](https://www.influxdata.com/) | Time-series storage |
| [Grafana](https://grafana.com/) | Dashboards |
| Docker Compose | Reproducible deployment |
| [gcx](https://github.com/grafana/gcx) | Syncs Grafana dashboards to and from the repo |

Runs out of the box with a built-in data simulator (three fake machines) so
you can see the whole pipeline working before connecting real hardware.

## Quick start

Requires Docker and Docker Compose.

```bash
cp .env.example .env
```

Generate real secrets instead of the placeholders (prints values to paste into `.env`):

```bash
make secrets
```

At minimum, set in `.env`:
- `INFLUXDB_INIT_ADMIN_TOKEN` and `INFLUXDB_TOKEN` — **use the same value for both.** The former is what InfluxDB creates itself with on first boot; the latter is what Node-RED/Grafana authenticate with.
- `INFLUXDB_INIT_PASSWORD`, `GRAFANA_ADMIN_PASSWORD` — real passwords.
- `NODE_RED_CREDENTIAL_SECRET` — random string, used to encrypt any credentials Node-RED stores.
- `INFLUXDB_INIT_ORG` / `INFLUXDB_ORG` and `INFLUXDB_INIT_BUCKET` / `INFLUXDB_BUCKET` — keep each pair identical; rename to match the client. If you rename the org, also update `INFLUX_ORG` in the `Makefile` (used by `make sync-influx` / `make restore-influx`).

Then bring up the stack:

```bash
make up
# or: docker compose up -d --build
```

Open:
- Node-RED editor: http://localhost:1880
- Grafana: http://localhost:3000 (login with `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASSWORD`)
- InfluxDB UI: http://localhost:8086 (only if you keep the port binding in `docker-compose.yml`)

The **Plant Floor Overview** dashboard in Grafana should show data moving
within ~10 seconds — the simulator injects a reading every 3 seconds for one
of three fake machines (`press-01`, `press-02`, `cnc-04`).

## Repo layout

```
.
├── docker-compose.yml              # Core 3-service stack
├── docker-compose.portainer.yml    # Optional: add a Docker GUI (see file header)
├── .env.example                    # Copy to .env, fill in secrets
├── Makefile                        # up / down / logs / backup / secrets / sync / restore (see below)
├── node-red/
│   ├── Dockerfile                  # Bakes in node-red-contrib-influxdb at build time
│   └── data/
│       ├── flows.json              # Repo copy of the pipeline — see "Saving your work" below
│       ├── settings.js             # Repo copy of runtime config (no secrets hardcoded)
│       └── package.json            # Palette manifest
├── influxdb/
│   └── template.yml                # Buckets, tasks, etc. as an InfluxDB template (make sync-influx)
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/influxdb.yml    # Auto-configures the InfluxDB datasource
│   │   └── dashboards/dashboard.yml    # Auto-loads dashboards from grafana/dashboards/
│   ├── dashboards/
│   │   └── plant-floor-overview.json   # Starter dashboard (provisioned)
│   └── resources/                      # Dashboards + folders exported by gcx (make sync-grafana)
└── docs/
    └── ARCHITECTURE.md             # Pipeline design + how to connect real equipment
```

**Note on `node-red/data/`:** these files are *not* bind-mounted into the
running container. Node-RED's actual runtime state (`flows.json`,
`settings.js`, `flows_cred.json`, `node_modules`, etc.) lives in a Docker
named volume (`node-red-data`), independent of this folder. The copies in
`node-red/data/` are what you see in git — they only reflect the live state
after you explicitly run `make sync-flows` (see below). A named volume was
chosen over a bind mount after hitting host-filesystem issues (notably
Docker Desktop on macOS interfering with Node-RED's atomic file rename on
deploy) that don't reproduce with this approach.

## Common tasks

```bash
make logs             # tail all container logs
make ps               # container status
make config           # validate docker-compose.yml + .env interpolation
make backup           # dump InfluxDB data + Node-RED flows/creds to ./backups/<timestamp>/
make backup-influx    # InfluxDB time-series data only, to ./backups/influx-<timestamp>/

make sync             # sync-flows + sync-grafana + sync-influx (running stack → repo)
make restore          # restore-flows + restore-grafana + restore-influx (repo → running stack)

make sync-flows       # Node-RED flows.json + settings.js: container → repo
make restore-flows    # Node-RED flows.json + settings.js: repo → container (then Deploy)
make sync-grafana     # Grafana dashboards + folders: Grafana → grafana/resources/ (gcx pull)
make restore-grafana  # Grafana dashboards + folders: grafana/resources/ → Grafana (gcx push)
make sync-influx      # InfluxDB buckets/tasks/etc.: container → influxdb/template.yml
make restore-influx   # InfluxDB buckets/tasks/etc.: influxdb/template.yml → container (fresh instance only)

make down             # stop the stack (keeps data — named volumes persist)
make clean            # stop the stack AND delete all volumes (destructive, commented out by default)
```

## Saving your work

Each service writes its live state to a Docker volume, not to this repo
folder. To get your changes into git, or to load a client's version onto a
fresh machine, use the sync/restore commands rather than editing repo files
directly and expecting them to take effect.

All sync/restore commands require the stack to already be running
(`make up`).

**While actively developing (UI → repo):**
```bash
# 1. Make changes in the UIs as normal:
#    - Node-RED (http://localhost:1880): edit flows, hit Deploy
#    - Grafana (http://localhost:3000): edit dashboards, hit Save dashboard
#    - InfluxDB (http://localhost:8086): add buckets, tasks, etc.
# 2. Pull the current state of all three into the repo
make sync
# 3. Review and commit
git add node-red/data/ grafana/resources/ influxdb/template.yml
git commit -m "Update pipeline for <change>"
```

**On a fresh clone or a new machine (repo → UI):**
```bash
make up                # creates the volumes + starts the containers
make restore-flows     # pushes the repo's flows.json + settings.js into Node-RED
# Open the Node-RED editor and hit Deploy so Node-RED picks up the change
make restore-grafana   # pushes committed dashboards into Grafana (needs gcx set up, see below)
```

Run `make restore-influx` only if `influxdb/template.yml` holds resources
beyond what InfluxDB creates on first boot from `.env` (extra buckets,
tasks, etc.) — see the InfluxDB section below.

### Node-RED: flows.json and settings.js

`sync-flows` and `restore-flows` operate via `docker cp` against the live
container, not the volume directly. After `restore-flows`, open the editor
and hit Deploy.

**`flows_cred.json` is intentionally excluded from this workflow.** It never
syncs to the repo and is not tracked in git. It lives only in the named
volume, decrypted using `NODE_RED_CREDENTIAL_SECRET` from `.env`. Re-enter
credentials (PLC auth, tokens, etc.) through the Node-RED editor on each new
deployment. If you need a recovery copy, `make backup` captures it as part
of the encrypted-at-rest backup — outside of git entirely.

### Grafana: dashboards via gcx

`sync-grafana` and `restore-grafana` use [gcx](https://github.com/grafana/gcx),
Grafana's CLI, which runs on the machine where you run `make` and talks to
Grafana over its API. It requires **Grafana 12 or later** (13+ fully
supported). Set it up once per machine, and once per client instance:

1. **Install gcx**
   ```bash
   brew install gcx
   # or: curl -fsSL https://raw.githubusercontent.com/grafana/gcx/main/scripts/install.sh | sh
   gcx --version
   ```
2. **Create a service account** in Grafana: Administration → Users and access → Service accounts → Add service account. Name it `gcx-sync`, role **Editor** (enough to pull and push dashboards) or **Admin** (also sees dashboards with restricted permissions — see below).
3. **Generate a token** on the service account's page: Add service account token → Generate token. Copy it immediately; Grafana only shows it once. Tokens start with `glsa_`.
4. **Log gcx in** to that Grafana instance, using a context name per client:
   ```bash
   gcx login acme --server http://<grafana-host>:3000 --token glsa_... --yes
   gcx config check
   ```
   `--server` must be the same address you use for that Grafana in your browser.

The Makefile uses gcx's **current** context. When you work on more than one
client, switch before syncing:
```bash
gcx config use-context acme
make sync-grafana
```

gcx writes its own resource format (dashboard JSON wrapped in
`apiVersion`/`kind`/`metadata`/`spec`) into
`grafana/resources/dashboards.<version>.dashboard.grafana.app/<uid>.json`.
The version in the folder name (e.g. `v0alpha1`) can differ from the
`apiVersion` inside the file (e.g. `dashboard.grafana.app/v2`) — that's
expected; `restore-grafana` pushes each file in its own format.

**Gotchas:**
- **Restricted dashboards are skipped silently.** If a dashboard has its own permissions that leave out the service account, gcx can't see it and won't pull it — no error. Add the service account under that dashboard's Settings → Permissions (Edit), or give the service account the Admin role.
- **Provisioned dashboards revert UI edits.** Dashboards loaded from `grafana/dashboards/` by `grafana/provisioning/dashboards/dashboard.yml` (like Plant Floor Overview) treat the JSON file as the source of truth — edits saved in the Grafana UI get overwritten when the file reloads. To change a provisioned dashboard, edit its JSON file and restart Grafana. If a client's dashboards will be edited in the UI, remove the dashboard provider file and the `./grafana/dashboards` mount and let gcx own the dashboards instead (pull once first, then `restore-grafana` brings back anything that disappears).
- **Keep gcx output out of `grafana/dashboards/`.** That folder is mounted for provisioning, which expects plain dashboard JSON; gcx's resource format doesn't belong there.
- **Data sources aren't synced.** Only dashboards and folders are. Data sources come from `grafana/provisioning/datasources/`; anything added through the Grafana UI won't survive a rebuild unless you add it there.

### InfluxDB: resource template

`sync-influx` runs `influx export all` inside the InfluxDB container and
writes every templatable resource in the org — buckets, tasks, Telegraf
configs, dashboards, labels — to `influxdb/template.yml`. It contains **no
tokens and no time-series data**, so it's safe to commit. The org name comes
from `INFLUX_ORG` in the `Makefile`; keep it matching `INFLUXDB_ORG` in `.env`.

`restore-influx` runs `influx apply` against that file. It's meant for a
**fresh instance**: on an instance that already has these resources (e.g.
the bucket created on first boot from `.env`), it can fail on name conflicts
or create duplicates.

Time-series data is never in git. Back it up with `make backup-influx` or
`make backup` (both write to `./backups/`, which is gitignored).

## Version control per client

Per the standard workflow: **one private repo per client.**

```bash
# Starting a new client from this template
git clone <this-repo-url> client-acme-pipeline
cd client-acme-pipeline
rm -rf .git && git init
git remote add origin <new-private-repo-url>
cp .env.example .env   # fill in this client's real secrets — .env is gitignored
# update INFLUX_ORG in the Makefile if you renamed the org in .env
make up
make restore-flows     # load the template's starter flows.json/settings.js into the new container
# set up a gcx context for this client's Grafana (see "Grafana: dashboards via gcx")
git add .
git commit -m "Initial pipeline for Acme Manufacturing"
git push -u origin main
```

Keep `.env`, `node-red/data/flows_cred.json`, and `backups/` out of every
commit (already covered by `.gitignore`) — they hold the InfluxDB token,
Grafana admin password, any credentials Node-RED encrypts, and raw data.
The Grafana service account token lives in gcx's own config
(`~/.config/gcx/config.yaml`) on your machine, not in the repo.

## Going live at a client site

1. Replace the simulator with a real input node for their equipment (MQTT / Modbus / OPC-UA / S7 / REST) — see `docs/ARCHITECTURE.md`.
2. Tune calibration constants, thresholds, and health-score weighting in the `Math`, `Stateful Ops`, and `Synthetic` function nodes for their actual machines, then `make sync-flows` and commit.
3. Build or adjust the client's dashboards in Grafana, then `make sync-grafana` and commit.
4. Set `NODE_RED_ENABLE_AUTH=true` and configure `adminAuth` in `node-red/data/settings.js` if the Node-RED editor is reachable outside a VPN/Tailscale network — edit via the Node-RED UI or a direct volume edit, then `make sync-flows` to capture it in git.
5. Turn off the public InfluxDB port binding in `docker-compose.yml` (leave InfluxDB reachable only on the internal Docker network) once you don't need it for debugging.
6. Decide on hosting (client-hosted vs. your managed VM vs. hybrid) and remote access (Tailscale vs. WireGuard) per the standard tool-stack decision tree.
7. Point Uptime Kuma (run centrally, across all clients) at this stack's exposed endpoints.
8. Set a real retention policy (`INFLUXDB_INIT_RETENTION`) matching what the client needs and what your storage budget allows, then `make sync-influx` and commit.

## Troubleshooting

**`docker compose up -d` (or `--build`) fails during the Node-RED image build with `npm ERR! code EAI_AGAIN` / `getaddrinfo EAI_AGAIN registry.npmjs.org`**

This is a DNS resolution failure inside the Docker build environment, not an
npm or Node-RED problem — the build container can't resolve
`registry.npmjs.org` (or any hostname) to an IP address at all, so the
`npm install` step in `node-red/Dockerfile` fails before it can fetch
`node-red-contrib-influxdb`.

**Fix:** point the Docker daemon at a known-good public DNS resolver
explicitly, rather than relying on whatever it auto-detected:

```bash
sudo tee /etc/docker/daemon.json <<EOF
{
  "dns": ["8.8.8.8"]
}
EOF
sudo systemctl restart docker
```

Then retry:

```bash
docker compose up -d --build
```

If you want to sanity-check the fix took effect before rebuilding the whole
stack, run a throwaway container against the same lookup that was failing:

```bash
docker run --rm busybox nslookup registry.npmjs.org
```

## Notes

- **Pin the InfluxDB image to 2.x**: the Docker `latest` tag for InfluxDB now points to InfluxDB 3, which doesn't support Flux. Keep `docker-compose.yml` on a specific 2.x tag (e.g. `influxdb:2.7`).
- **Environment variable substitution**: both Node-RED (`${VAR}` in `flows.json`) and Grafana (`$__env{VAR}` in provisioning YAML) resolve these directly from the container's environment at startup — nothing needs templating at build time. Update `.env` and restart the affected container to change them.
- **Why a named volume instead of a bind mount for `node-red/data/`**: bind-mounting `flows.json` directly triggered `EBUSY: resource busy or locked` errors on deploy, traced to macOS Spotlight indexing interfering with Node-RED's atomic rename-on-save. The named volume sidesteps this entirely; `make sync-flows` / `make restore-flows` bridge it back to git deliberately rather than automatically.

## To Do List

- Create separate InfluxDB tokens for write-only access (for Node-RED) and read-only access (for Grafana)
