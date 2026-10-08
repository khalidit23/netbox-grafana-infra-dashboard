# NetBox + Grafana Infrastructure Dashboard

Visualize your **Datacenter → Proxmox Cluster → Node → VM (with IP)** hierarchy in Grafana, using **NetBox** as the source of truth. No agents, no exporters: Grafana reads the NetBox PostgreSQL database directly (read-only).

> Built for sysadmins who want to answer one question quickly: *"Which VM is on which node, in which cluster, in which datacenter, and what is its IP?"*

## What you get

| Panel | Description |
|---|---|
| Stat cards | Total datacenters, clusters, nodes, VMs |
| Hierarchy cards | Cluster (orange) → Node (cyan) → VM (green) with icons and IP addresses |
| Node graph (alt. version) | Classic tree graph using Grafana's built-in Node Graph panel |
| Donut chart | VMs per cluster |
| Bar gauge | VMs per node |
| Geomap | Datacenter locations on OpenStreetMap |
| Table | Cluster / Node / VM / IP list |

## Architecture

```
Proxmox clusters (manual entry or CSV import)
            │
            ▼
        NetBox (DCIM)  ──►  PostgreSQL (read-only user: grafana_ro)
                                        │
                                        ▼
                                    Grafana  ──►  Dashboards
```

Data model used in NetBox:

```
Site (datacenter)
 └── Cluster (Proxmox cluster)
      └── Device (Proxmox node, role: Hypervisor)
           └── Virtual Machine (+ IP address)
```

## Requirements

- Linux server with **Docker** and **Docker Compose v2**
- An existing **Grafana** container (tested with Grafana 12/13)
- ~2 GB free RAM for NetBox (NetBox + PostgreSQL + 2x Valkey/Redis)
- Internet access from the Grafana container (to install one panel plugin)

Tested on CentOS 7 (kernel 3.10), Docker 26, Docker Compose 2.27. See the [CentOS 7 notes](#centos-7--old-kernel-notes).

---

## Step 1: Install NetBox (Docker)

```bash
cd /opt
sudo git clone -b release https://github.com/netbox-community/netbox-docker.git
cd netbox-docker
```

Create the override file (change NetBox port to 8000):

```bash
sudo tee docker-compose.override.yml <<'EOF'
services:
  netbox:
    ports:
      - "8000:8080"
  postgres:
    image: docker.io/postgres:16
EOF
```

Start it:

```bash
sudo docker compose pull
sudo docker compose up -d
sudo docker compose logs -f netbox      # wait for migrations to finish
```

Create the admin user and open the firewall:

```bash
sudo docker compose exec netbox /opt/netbox/netbox/manage.py createsuperuser
sudo firewall-cmd --permanent --add-port=8000/tcp && sudo firewall-cmd --reload
```

Open `http://SERVER_IP:8000`.

> **Change default passwords** in `env/postgres.env` and `env/netbox.env` before the first `up` if this server is reachable by others.

### CentOS 7 / old kernel notes

If PostgreSQL exits immediately with:

```
could not write to file "postmaster.pid": Operation not permitted
```

the host's old `libseccomp` blocks syscalls used by newer images. Fixes, in order:

1. Use a Debian-based Postgres image (`postgres:16`) instead of `postgres:18-alpine` (already in the override above).
2. If it still fails, disable the seccomp profile for the postgres container only:

```yaml
  postgres:
    image: docker.io/postgres:16
    security_opt:
      - seccomp:unconfined
```

Then `sudo docker compose down -v && sudo docker compose up -d`.

CentOS 7 is end-of-life. Plan a migration to Rocky Linux 9 / AlmaLinux 9.

---

## Step 2: Enter your data in NetBox

Create objects in this order (each depends on the previous):

1. **Organization → Sites**: one per datacenter. Fill **Latitude / Longitude** (needed for the map; max 6 decimals).
2. **Devices → Manufacturers**: e.g. `Dell`
3. **Devices → Device Types**: e.g. `R640`
4. **Devices → Device Roles**: e.g. `Hypervisor` (leave "VM Role" unticked)
5. **Virtualization → Cluster Types**: `Proxmox`
6. **Virtualization → Clusters**: e.g. `dccl`, `drcl` (set Scope = Site)
7. **Devices → Devices**: the Proxmox nodes, **with Cluster set** (required, otherwise you cannot assign VMs to the node)
8. **Virtualization → Virtual Machines**: with Cluster **and Device** set
9. **IP addresses**: assign an IP to each VM (set it as the VM's **Primary IPv4**, or assign it to a VM interface)

### Bulk import with CSV

**Devices → Devices → Import**

```csv
name,role,manufacturer,device_type,site,status,cluster
dc101,Hypervisor,Dell,R640,DC-Primary,active,dccl
dc102,Hypervisor,Dell,R640,DC-Primary,active,dccl
dr101,Hypervisor,Dell,R640,DC-DR,active,drcl
dr102,Hypervisor,Dell,R640,DC-DR,active,drcl
```

**Virtualization → Virtual Machines → Import**

```csv
name,status,site,cluster,device,vcpus,memory,disk
web-01,active,DC-Primary,dccl,dc101,4,8192,100
db-01,active,DC-Primary,dccl,dc101,8,16384,500
web-02,active,DC-DR,drcl,dr101,4,8192,100
```

`memory` is in MB, `disk` in GB. `cluster` and `device` must match (a VM on `dc101` must use cluster `dccl`).

### Tip: list VMs from Proxmox

Run on any node of a cluster to get all VMs, then convert to CSV:

```bash
pvesh get /cluster/resources --type vm --output-format json
```

---

## Step 3: Connect Grafana to the NetBox database

### 3.1 Put Grafana on the NetBox Docker network

```bash
sudo docker network connect netbox-docker_default grafana
```

(`grafana` is your Grafana container name; check with `docker ps`.)

### 3.2 Create a read-only database user

```bash
cd /opt/netbox-docker
sudo docker compose exec postgres psql -U netbox -d netbox
```

```sql
CREATE USER grafana_ro WITH PASSWORD 'CHANGE_ME_STRONG_PASSWORD';
GRANT CONNECT ON DATABASE netbox TO grafana_ro;
GRANT USAGE ON SCHEMA public TO grafana_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO grafana_ro;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO grafana_ro;
\q
```

### 3.3 Add the data source in Grafana

**Connections → Data sources → Add → PostgreSQL**

| Field | Value |
|---|---|
| Host URL | `netbox-docker-postgres-1:5432` |
| Database | `netbox` |
| Username | `grafana_ro` |
| Password | your password |
| TLS/SSL Mode | `disable` |

Click **Save & test**. You should see *Database Connection OK*.

> Using `SERVER_IP:5432` gives `connection refused` because the Postgres port is not published to the host. Use the container name instead (this also keeps the DB off the network).
>
> `docker network connect` is lost if the Grafana container is recreated. Add the network to your Grafana compose file / run command to make it permanent.

---

## Step 4: Install the panel plugin

The icon-style hierarchy uses the **Business Text** (Dynamic Text) panel.

```bash
sudo docker exec -it grafana grafana cli plugins install marcusolsson-dynamictext-panel
sudo docker restart grafana
```

> `volkovlabs-text-panel` returned `404 Plugin not found` on Grafana 13.1.1. `marcusolsson-dynamictext-panel` worked.
>
> If you do not want any plugin, use the Node Graph version of the dashboard (`netbox-infra-dashboard-pro.json`).

---

## Step 5: Import the dashboards

**Dashboards → New → Import → Upload JSON file**, then pick your NetBox PostgreSQL data source in the **NetBox DB** dropdown at the top.

| File | Panels | Plugin needed |
|---|---|---|
| `dashboards/netbox-infra-dashboard.json` | Icon cards with IPs, stats, donut, bar gauge, map, table | `marcusolsson-dynamictext-panel` |

Dashboards auto-refresh every minute, so changes in NetBox appear without any action.

---

## How the queries work

The hierarchy panel builds its HTML directly in SQL (one row, one `html` column) and the panel template renders it:

```
{{{data.[0].html}}}
```

The VM IP is taken from the VM's **Primary IPv4**. If it is not set, the first IP assigned to any VM interface is used. If there is none, `no IP` is shown.

Main NetBox tables used: `dcim_site`, `dcim_device`, `virtualization_cluster`, `virtualization_virtualmachine`, `virtualization_vminterface`, `ipam_ipaddress`. Table and column names can change between NetBox versions; if a query fails, check them with `\dt` in `psql`.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `netbox` container `unhealthy`, log says `failed to resolve host 'postgres'` | Postgres crashed. Check `docker compose logs postgres` (see CentOS 7 notes) |
| Grafana: `connect: connection refused` | Use `netbox-docker-postgres-1:5432`, and make sure Grafana is on the `netbox-docker_default` network |
| Map shows nothing | Set Latitude/Longitude on the Sites in NetBox |
| Map shows `API KEY REQUIRED` watermark | Change the basemap layer from CARTO to OpenStreetMap |
| VM list empty / node has no VMs | Set both **Cluster** and **Device** on each VM |
| Cannot select a node when adding a VM | The device must belong to the same Cluster |
| `Panel plugin not found` | Install the plugin (Step 4) and restart Grafana |
| Panel shows raw/empty text | In panel edit: Render template = `All rows`, Content = `<div class="infra">{{{data.[0].html}}}</div>` |
| IP shows `no IP` | Set the VM's Primary IPv4, or assign an IP to a VM interface |
| `relation ... does not exist` | Table names differ in your NetBox version; check with `\dt` |
| "Add visualization" button missing | New Grafana UI: use the blue `+` on the right or the `+` at top-left |

---

## Security notes

- Keep the PostgreSQL port unpublished. Grafana reaches it over the Docker network.
- Use a strong password for `grafana_ro`; it only has `SELECT`.
- Change the default NetBox/Postgres passwords and `SECRET_KEY` in `env/*.env`.
- Put NetBox and Grafana behind a reverse proxy with HTTPS if exposed beyond your LAN.

## Repository layout

```
.
├── README.md
└── dashboards/
    ├── netbox-infra-dashboard.json
```

## Credits

- [NetBox](https://github.com/netbox-community/netbox) and [netbox-docker](https://github.com/netbox-community/netbox-docker)
- [Grafana](https://grafana.com)
- [Business Text / Dynamic Text panel](https://grafana.com/grafana/plugins/marcusolsson-dynamictext-panel/)

## License

MIT
