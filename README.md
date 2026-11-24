# Identiqa NMS — Zabbix Docker Stack (with License Services)

Docker Compose stack that runs a branded **Zabbix 7.4** server ("Identiqa NMS")
together with the Identiqa **license agent** and **license proxy**. The web UI is
reachable only while a valid license is present.

## Services

| Service | Image / build | Port | Purpose |
|---|---|---|---|
| `mysql-server` | `mysql:8.0` | internal | Zabbix database |
| `zabbix-server` | `zabbix/zabbix-server-mysql:7.4-alpine-latest` | `10051` | Zabbix server |
| `zabbix-web` | `zabbix/zabbix-web-nginx-mysql:7.4-alpine-latest` | internal `8080` | Zabbix frontend with Identiqa branding |
| `zabbix-agent` | `zabbix/zabbix-agent:alpine-7.4-latest` | `10050` | Agent monitoring the stack itself |
| `zabbix-snmptraps` | `zabbix/zabbix-snmptraps:latest` | `162/udp` | SNMP trap receiver |
| `license-agent` | `./license-agent` | — | Checks the license every 10 minutes, starts/stops `zabbix-server`, reports host count and version |
| `license-proxy` | `./license-proxy` | **`8080`** → 3333 | Public entry point: validates the license on each request, then proxies to `zabbix-web`; otherwise shows `custom.html` |

## How licensing works

1. `license/license.lic` holds two lines: the **license key** and the **instance ID**.
2. `license-proxy` checks the key against the Identiqa license API before
   forwarding any request to the Zabbix web UI.
3. `license-agent` (which has access to the Docker socket) re-checks every
   10 minutes. It stops `zabbix-server` if the license is invalid and starts it
   again when it is valid. It also reports host count and Zabbix version to the
   license backend (see the `license_backend` repository).

## Requirements

- Docker Engine with the Compose plugin
- Outbound HTTPS from the host to the license API
- A license key and instance ID from Identiqa

## Quick start

```bash
git clone https://github.com/identiqalabs/zabbix-docker.git
cd zabbix-docker

# 1. Add your license (key on line 1, instance ID on line 2)
printf '%s\n%s\n' "<LICENSE_KEY>" "<INSTANCE_ID>" > license/license.lic

# 2. Start the stack
docker compose up -d --build

# 3. Check that everything is running
docker compose ps
```

Open `http://<server-ip>:8080`. The default Zabbix login is `Admin` / `zabbix`;
**change it immediately.**

## Configuration

- **Database passwords** are set in `docker-compose.yaml`
  (`MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD`). Change them before the first start,
  and update every service that uses them.
- **Time zone**: `TZ` / `PHP_TZ` (default `Asia/Kolkata`).
- **Branding**: files in `zabbix-custom-ui/` (logo, favicon, `brand.conf.php`,
  theme) are mounted into `zabbix-web`.
- **SNMP traps**: `zabbix-custom-ui/snmptrapd.conf`.

## Useful commands

```bash
docker compose logs -f license-agent     # license check activity
docker compose restart zabbix-server
docker compose down                      # stop (data kept in volumes)
```

## Related repositories

- `zabbix-docker1` — newer setup notes for this stack, including OpenStack VMs
- `zbxdocker` — variant where the license agent runs on the host instead of in a container
- `license_backend` — license API used by the agent and proxy
- `OVA-SETUP` — preparing a client VM / OVA that prompts for the license on first login
