# Lab Inventory — rasp5 (.85) + Rocky lab (.98–.102)
**Updated:** 2026-09-19 | Rocky Linux 9.8 VMs (Hyper-V) + Raspberry Pi 5 control node | All hosts verified live via SSH

---

## Network & Service Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              192.168.1.x Home Lab                                      │
│                                                                                        │
│  .85 rasp5  (Raspberry Pi 5 · Ubuntu 24.04.5 · aarch64)                                │
│  ┌────────────────────────────────────────────────────────────────────────┐          │
│  │ CONTROL / MGMT NODE — Claude Code + opencode run here; holds inventory   │          │
│  │  [Docker]  twingate-jade-limpet  (Twingate connector, remote access)     │          │
│  │  node_exporter :9100   glances :61209 (localhost)   atop               │
  │  alloy→Loki (log shipper)                                               │          │
│  └────────────────────────────────────────────────────────────────────────┘          │
│                                                                                        │
│  .98 hv-rocky-linux-1                          .99 hv-rocky-linux-2                    │
│  ┌───────────────────────────────────────┐    ┌────────────────────────────────────┐ │
│  │ ★ OBSERVABILITY HUB [Docker Compose]   │    │ k3s v1.36.3+k3s1  (single node)    │ │
│  │   grafana        :3000                 │    │   ingress-nginx  :80 :443 :8443    │ │
│  │   prometheus     :9090                 │    │   coredns / metrics-server /       │ │
│  │   alertmanager   :9093                 │    │   local-path-provisioner           │ │
│  │   loki           :3100                 │    │   k3s API        :6443             │ │
│  │   cadvisor       :8081                 │    │  Workloads (NodePort):             │ │
│  │ [Docker]                               │    │   nginx        :30453              │ │
│  │   vnotes         :8000  (VulnNotes API)│    │   juice-shop   :30300              │ │
│  │ node_exporter :9100 · alloy            │    │   vampi        :30500              │ │
│  │ mcp-heartbeat svc · mcp-sweep svc      │    │ cloudflared (Cloudflare Tunnel)    │ │
│  └───────────────────────────────────────┘    │ node_exporter :9100 · alloy        │ │
│                                                └────────────────────────────────────┘ │
│  .100 hv-rocky-linux-3                         .101 hv-rocky-linux-4                    │
│  ┌───────────────────────────────────────┐    ┌────────────────────────────────────┐ │
│  │ [Podman] kong:3.6  :80 :8001           │    │ [Podman rootful] crAPI stack:      │ │
│  │ [Docker] jenkins   :8080 :50000        │    │   postgresdb  :5432 (internal)     │ │
│  │          jenkins-docker(dind) :2376    │    │   mongodb     :27017 (internal)    │ │
│  │          cadvisor  :8081               │    │   chromadb    :8000 (internal)     │ │
│  │ node_exporter :9100 · podman-exp :9882 │    │   mailhog     :8025                │ │
│  │ alloy                                  │    │   gateway-svc :443 (internal)      │ │
│  └───────────────────────────────────────┘    │   crapi-identity :8080/:8989       │ │
│                                                │   crapi-community :6060            │ │
│  .102 hv-rocky-linux-5                         │   crapi-workshop                   │ │
│  ┌───────────────────────────────────────┐    │   crapi-chatbot  :5500            │ │
│  │ noname-sensor svc (eBPF)               │    │   crapi-web  :8888/:8443           │ │
│  │ crapi-mcp   svc :8009                  │    │             :30080/:30443          │ │
│  │ noname-mcp  svc :8013                  │    │   searxng     :8080                │ │
│  │ ai-sim      svc :8011(LLM) :8012(GenAI)│    │   juice-shop  :3000                │ │
│  │ [Podman] vampi-mcp :5000               │    │ node_exporter :9100 · podman :9882 │ │
│  │ node_exporter :9100 · podman-exp :9882 │    │ noname-sensor svc · alloy          │ │
│  │ alloy                                  │    │                                    │ │
│  └───────────────────────────────────────┘    └────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Observability / Monitoring Topology

Installed 2026-09-18 (metrics) and 2026-09-19 (logs). Full runbook: `~/MONITORING_SETUP.md`
on rasp5; helper scripts in `~/monitoring/`.

```
                         ┌──────────────────────────────────────────┐
                         │  .98  /opt/monitoring  (Docker Compose,   │
                         │        all network_mode: host)            │
   metrics  ──scrape──►  │  Prometheus :9090 ──► Alertmanager :9093  │
   (pull)                │        ▲                                  │
                         │        │ remote-write (log-derived series)│
   logs     ──push────►  │  Loki :3100  ──ruler──► Alertmanager      │
   (Alloy)               │        │                                  │
                         │  Grafana :3000 ◄── queries both           │
                         │  cAdvisor :8081 (local container metrics) │
                         └──────────────────────────────────────────┘
        ▲  metrics targets                         ▲  log sources (Alloy agent)
        │                                           │
  node_exporter :9100  → .98 .99 .100 .101 .102 .85 Alloy on .98 .99 .100 .101 .102 + .85
  cAdvisor      :8081  → .98 .100                    - common.alloy: journal + auditd (all)
  podman-exp    :9882  → .100 .101 .102              - docker.alloy: .98 .100 (+ .85 docker)
  k3s kubelet   :10250 → .99 (SA-token, TLS-verified)- k3s-pods.alloy: .99
```

**Notes**
- Grafana admin password lives in `/opt/monitoring/.env`.
- Prometheus config is a mounted **directory** (`prometheus/config/`); reload with
  `curl -XPOST localhost:9090/-/reload`. Runs with `--web.enable-remote-write-receiver`.
- Loki logs firewall-allowed only from .99–.102 (and .85). Loki ruler rules:
  `/opt/monitoring/loki/rules/fake/lab-logs.yml`; Prometheus baseline rules: `rules/logs.yml`.
- Alertmanager receiver is still `null` — alerts fire but are **not delivered anywhere yet**.
- Dashboards are generated by `gen_dashboards.py` and file-provisioned — edit the script, not the UI.
- Firewalls: .98/.101/.102 use firewalld rich rules (**never** `firewall-cmd --reload` there);
  .99/.100 use a standalone nft table `inet monitoring_acl` via `monitoring-acl.service`.
- **TODO:** rk1–rk4 (.75–.78) log shipping still pending (were offline 2026-09-19).
  .103–.105 deliberately excluded from monitoring.

---

## Host Summary

| IP | Hostname | Role | OS | RAM | Disk Used | Uptime |
|----|----------|------|-----|-----|-----------|--------|
| .85 | rasp5 | Control node (Claude Code/opencode), Twingate, monitoring source | Ubuntu 24.04.5 (RPi 5) | 7.7 Gi | 16 G / 117 G (14%) | 15h |
| .98 | hv-rocky-linux-1 | **Observability hub** + VulnNotes API + MCP session host | Rocky 9.8 | 8.6 Gi | 16 G / 70 G (22%) | 22d |
| .99 | hv-rocky-linux-2 | k3s cluster + Cloudflare ingress | Rocky 9.8 | 6.0 Gi | 15 G / 70 G (21%) | 31d |
| .100 | hv-rocky-linux-3 | Kong API Gateway + Jenkins CI | Rocky 9.8 | 8.4 Gi | 15 G / 70 G (21%) | 31d |
| .101 | hv-rocky-linux-4 | crAPI lab + Noname Sensor | Rocky 9.8 | 8.1 Gi | 18 G / 70 G (25%) | 31d |
| .102 | hv-rocky-linux-5 | Noname Sensor + MCP servers + AI sim | Rocky 9.8 | 3.9 Gi | 20 G / 70 G (29%) | 31d |

Rocky VMs: Intel i9-9900K @ 3.60 GHz | Rocky Linux 9.8 (Blue Onyx) | Hyper-V.
Control node: Raspberry Pi 5 Model B Rev 1.0 | aarch64 | Ubuntu 24.04.5 LTS.

---

## .85 — rasp5

**Role:** Control / management node — Claude Code and opencode run here; holds this
inventory repo, the monitoring runbook (`~/MONITORING_SETUP.md`) and staging (`~/monitoring/`).

### Running Services

| Service | Description | Status |
|---------|-------------|--------|
| `docker.service` | Docker runtime | Running |
| `node_exporter.service` | Prometheus node exporter (:9100) — scraped by Prometheus on .98 | Running |
| `alloy.service` | Grafana Alloy — ships journal + docker logs to Loki on .98 | Running |
| `glances.service` | Glances system monitor (`127.0.0.1:61209`) | Running |
| `atop.service` / `atopacct.service` | atop resource logging + process accounting | Running |
| `cups.service` | Printing | Running |

### Docker Containers

| Name | Image | Status | Ports |
|------|-------|--------|-------|
| `twingate-jade-limpet` | `twingate/connector:1` | Up (healthy) | — (outbound tunnel) |

### Notes
- Log shipping added here 2026-09-19 (deb install) via `~/monitoring/alloy/install-alloy-ubuntu.sh`.
- node_exporter added 2026-09-19 (binary v1.12.1 arm64 at `/usr/local/bin`, runs as user
  `node_exporter`, systemd unit mirrors the Rocky fleet). Added to Prometheus `node` job on
  .98 with label `host: rasp5`; target confirmed **up**. rasp5 has no firewall (ufw inactive),
  so no ACL change was needed.
- SSH mesh hub: passwordless as `mcropsey` to the Rocky fleet `.98`–`.105` (`.103`–`.105`
  currently reject the key) using `~/.ssh/id_ed25519`.

---

## .98 — hv-rocky-linux-1

**Role:** Observability hub (Grafana/Prometheus/Loki/Alertmanager) + VulnNotes API lab target + MCP session host

### Systemd Services

| Service | Description | Status |
|---------|-------------|--------|
| `docker.service` | Docker CE container runtime | Running |
| `containerd.service` | containerd | Running |
| `node_exporter.service` | Prometheus node exporter (:9100) | Running |
| `alloy.service` | Grafana Alloy — journal + auditd + docker logs → Loki | Running |
| `mcp-heartbeat.service` | MCP heartbeat — long-lived session for Noname | Running |
| `mcp-sweep.service` | MCP full sweep — long-lived session, reads | Running |

### Docker Containers

| Name | Image | Status | Ports |
|------|-------|--------|-------|
| `grafana` | `grafana/grafana:13.2.2` | Up | `:3000` (host net) |
| `prometheus` | `prom/prometheus:v3.14.0` | Up | `:9090` (host net) |
| `alertmanager` | `prom/alertmanager:v0.34.1` | Up | `:9093` (host net) |
| `loki` | `grafana/loki:3.7.8` | Up | `:3100` (host net) |
| `cadvisor` | `ghcr.io/google/cadvisor:v0.60.6` | Up (healthy) | `:8081` (host net) |
| `vnotes` | `vnotes:latest` | Up (healthy) | `0.0.0.0:8000→8000` |

> **Change:** the VulnNotes app is now container `vnotes` (image `vnotes:latest`);
> the monitoring compose stack runs alongside it under `/opt/monitoring`.

### Key URLs

| URL | What |
|-----|------|
| http://192.168.1.98:3000 | **Grafana** (admin pw in `/opt/monitoring/.env`) |
| http://192.168.1.98:9090 | Prometheus |
| http://192.168.1.98:9093 | Alertmanager |
| http://192.168.1.98:3100 | Loki (API; query via Grafana) |
| http://192.168.1.98:8081 | cAdvisor |
| http://192.168.1.98:8000 | VulnNotes SPA — login, notes, BOLA Lab |
| http://192.168.1.98:8000/docs | Swagger UI |
| http://192.168.1.98:8000/openapi.json | OpenAPI 3.1 spec |
| http://192.168.1.98:8000/health | Health check |

### VulnNotes Demo Credentials

| Username | Password | Role |
|----------|----------|------|
| alice | alice123 | user |
| bob | bob12345 | user |
| charlie | charlie1 | user |
| admin | admin123 | admin |

### Lab Scripts (`~/notes-test/`)

| File | Purpose | Command |
|------|---------|---------|
| `normal_traffic.py` | Baseline API traffic (4 concurrent users) | `python3 normal_traffic.py --base-url http://192.168.1.98:8000 --duration 120` |
| `exploit_bola.py` | BOLA — cross-user note read/write/delete | `python3 exploit_bola.py --base-url http://192.168.1.98:8000 --aggressive` |
| `requirements.txt` | `httpx>=0.27.0` | `pip3 install -r requirements.txt` |

---

## .99 — hv-rocky-linux-2

**Role:** Kubernetes (k3s) cluster + Cloudflare Tunnel ingress

### Systemd Services

| Service | Description | Status |
|---------|-------------|--------|
| `k3s.service` | Lightweight Kubernetes **v1.36.3+k3s1** (control-plane + worker, Ready) | Running |
| `cloudflared.service` | Cloudflare Tunnel client | Running |
| `node_exporter.service` | Prometheus node exporter (:9100) | Running |
| `alloy.service` | Grafana Alloy — journal + auditd + k3s pod logs → Loki | Running |

### k3s Workloads

| Namespace / Deployment | Image | Service | External |
|------------------------|-------|---------|----------|
| `default/nginx` | `nginx` | NodePort | `:30453` |
| `juiceshop/juice-shop` | `bkimminich/juice-shop:latest` | NodePort | `:30300` |
| `vampi/vampi` | (noname-sensor 3.3.60 sidecar) | NodePort | `:30500` |
| `ingress-nginx/ingress-nginx-controller` | ingress-nginx | ClusterIP | :80/:443/:8443 |
| `kube-system/coredns` | mirrored-coredns 1.14.6 | ClusterIP | — |
| `kube-system/metrics-server` | mirrored-metrics-server v0.9.0 | ClusterIP | — |
| `kube-system/local-path-provisioner` | local-path-provisioner v0.0.36 | — | — |

### Listening Ports

| Port | Purpose |
|------|---------|
| 80 / 443 / 8181 / 8443 | ingress-nginx (HTTP/HTTPS/status/controller) |
| 6443 | k3s API server |
| 10250 | kubelet (Prometheus scrape target, SA-token auth, TLS-verified) |
| 30300 / 30453 / 30500 | NodePorts (juice-shop / nginx / vampi) |
| 9100 | node_exporter |

### Notes

- Single-node cluster — `hv-rocky-linux-2` is both control-plane and worker.
- Cloudflared routes external tunnel traffic into the k3s ingress.
- ⚠ `sudo` drops `/usr/local/bin` — call `sudo /usr/local/bin/k3s kubectl ...` by full path.
- No firewalld here — uses nft table `inet monitoring_acl` (`monitoring-acl.service`).

---

## .100 — hv-rocky-linux-3

**Role:** Kong API Gateway (Podman) + Jenkins CI/CD (Docker)

### Systemd Services

| Service | Description | Status |
|---------|-------------|--------|
| `docker.service` | Docker CE container runtime | Running |
| `containerd.service` | containerd | Running |
| `node_exporter.service` | Prometheus node exporter (:9100) | Running |
| `prometheus-podman-exporter.service` | Podman metrics exporter (:9882) | Running |
| `alloy.service` | Grafana Alloy — journal + auditd + docker logs → Loki | Running |

### Podman Containers (rootful)

| Name | Image | Status | Ports |
|------|-------|--------|-------|
| `kong` | `kong:3.6` | Up 4 weeks | `0.0.0.0:80→8000`, `0.0.0.0:8001→8001`, 8443-8444 |

### Docker Containers

| Name | Image | Status | Ports |
|------|-------|--------|-------|
| `jenkins` | `myjenkins-noname:2.568.3-jdk21` | Up | `0.0.0.0:8080→8080`, `0.0.0.0:50000→50000` |
| `jenkins-docker` | `docker:dind` | Up | `0.0.0.0:2376→2376` (2375 internal) |
| `cadvisor` | `ghcr.io/google/cadvisor:v0.60.6` | Up (healthy) | `:8081` (host net) |

> **Change:** Jenkins image is now pinned to `myjenkins-noname:2.568.3-jdk21` (was `:latest`);
> cAdvisor added as a second local container-metrics source for Prometheus.

### Key URLs

| URL | What |
|-----|------|
| http://192.168.1.100:8080 | Jenkins CI/CD web UI |
| http://192.168.1.100:8080/blue | Blue Ocean pipeline view |
| http://192.168.1.100:80 | Kong proxy (HTTP) |
| http://192.168.1.100:8001 | Kong Admin API |
| http://192.168.1.100:8081 | cAdvisor |

### Jenkins Details

| Item | Value |
|------|-------|
| Admin user | `admin` |
| Image | `myjenkins-noname:2.568.3-jdk21` — Jenkins + JDK21 + Docker CLI + pre-installed plugins |
| Plugins | blueocean, docker-workflow, github, github-branch-source, pipeline-model-definition, credentials-binding, plain-credentials, git, workflow-aggregator |
| GitHub credential ID | `github-token` (Classic PAT, Secret text) |
| DinD port | 2376 (TLS) |

### Notes
- No firewalld here — uses nft table `inet monitoring_acl` (`monitoring-acl.service`).

---

## .101 — hv-rocky-linux-4

**Role:** crAPI vulnerable app lab + Noname Sensor

### Systemd Services

| Service | Description | Status |
|---------|-------------|--------|
| `noname-sensor.service` | Noname Security API traffic sensor (eBPF) | Running |
| `node_exporter.service` | Prometheus node exporter (:9100) | Running |
| `prometheus-podman-exporter.service` | Podman metrics exporter (:9882) | Running |
| `alloy.service` | Grafana Alloy — journal + auditd → Loki | Running |

### Podman Containers (rootful)

| Name | Image | Status | Ports |
|------|-------|--------|-------|
| `postgresdb` | `postgres:14` | Up (healthy) | 5432 internal |
| `mongodb` | `mongo:4.4` | Up (healthy) | 27017 internal |
| `chromadb` | `chromadb/chroma:latest` | Up (healthy) | 8000 internal |
| `mailhog` | `crapi/mailhog:latest` | Up (healthy) | `0.0.0.0:8025→8025` (1025 internal) |
| `api.mypremiumdealership.com` | `crapi/gateway-service:latest` | Up (healthy) | 443 internal |
| `crapi-identity` | `crapi/crapi-identity:latest` | Up (healthy) | 8080, 8989, 10001 internal |
| `crapi-community` | `crapi/crapi-community:latest` | Up (healthy) | 6060 internal |
| `crapi-workshop` | `crapi/crapi-workshop:latest` | Up (healthy) | — |
| `crapi-chatbot` | `crapi/crapi-chatbot:latest` | Up | `0.0.0.0:5500→5500` (5002 internal) |
| `crapi-web` | `crapi/crapi-web:latest` | Up (healthy) | `:8888→80`, `:8443→443`, `:30080→80`, `:30443→443` |
| `searxng` | `searxng/searxng:latest` | Up 4 weeks | `0.0.0.0:8080→8080` |
| `juice-shop` | `bkimminich/juice-shop:latest` | Up 27h | `0.0.0.0:3000→3000` |

> **Change:** juice-shop is now **UP** (previously flagged DOWN in this doc).

### Key URLs

| URL | What |
|-----|------|
| http://192.168.1.101:8888 | crAPI web UI (HTTP) |
| https://192.168.1.101:8443 | crAPI web UI (HTTPS) |
| http://192.168.1.101:30080 | crAPI NodePort HTTP |
| https://192.168.1.101:30443 | crAPI NodePort HTTPS |
| http://192.168.1.101:8025 | MailHog (captures crAPI emails) |
| http://192.168.1.101:5500 | crAPI chatbot |
| http://192.168.1.101:8080 | SearXNG search |
| http://192.168.1.101:3000 | OWASP Juice Shop |

### Notes
- firewalld rich rules in use — **never** `firewall-cmd --reload` here (breaks container networking).

---

## .102 — hv-rocky-linux-5

**Role:** Noname Sensor + MCP servers (crAPI, Noname mgmt, VAmPI) + AI simulator

### Systemd Services

| Service | Description | Status | Port |
|---------|-------------|--------|------|
| `noname-sensor.service` | Noname Security API traffic sensor (eBPF) | Running | — |
| `crapi-mcp.service` | crAPI MCP Server (Streamable HTTP) | Running | 8009 |
| `noname-mcp.service` | Noname Management API MCP Server (Streamable HTTP) | Running | 8013 |
| `ai-sim.service` | AI-Sim — LLM + GenAI API simulator for sensor discovery | Running | 8011 (LLM), 8012 (GenAI) |
| `node_exporter.service` | Prometheus node exporter | Running | 9100 |
| `prometheus-podman-exporter.service` | Podman metrics exporter | Running | 9882 |
| `alloy.service` | Grafana Alloy — journal + auditd → Loki | Running | — |

### Podman Containers (rootful)

| Name | Image | Status | Ports |
|------|-------|--------|-------|
| `vampi-mcp` | `localhost/vampi-mcp:latest` | Up 2 weeks | `0.0.0.0:5000→5000` |

> **Change:** added `noname-mcp` (:8013), the `vampi-mcp` container (:5000), and confirmed
> `ai-sim` serves both LLM (:8011) and GenAI (:8012) from `/opt/ai-sim`.

### Key URLs

| URL | What |
|-----|------|
| http://192.168.1.102:8009 | crAPI MCP Server |
| http://192.168.1.102:8013 | Noname Management API MCP Server |
| http://192.168.1.102:8011 | AI-Sim LLM API |
| http://192.168.1.102:8012 | AI-Sim GenAI API |
| http://192.168.1.102:5000 | VAmPI MCP container |

### Notes
- firewalld rich rules in use — **never** `firewall-cmd --reload` here.
- Service definitions live under `/opt/{crapi-mcp,noname-mcp,ai-sim}` (each with its own `.venv`).

---

## Change Log

| Date | Host | Change |
|------|------|--------|
| 2026-09-19 | all | Full live re-verification via SSH; doc scope expanded to include rasp5 (.85). |
| 2026-09-19 | .85 | node_exporter v1.12.1 installed and added to Prometheus `node` job on .98 (target up) — control node now visible in metrics dashboards. |
| 2026-09-19 | .85 | Added as control node — Docker (twingate connector), Alloy log shipping, glances, atop. |
| 2026-09-19 | .98 | Observability hub deployed — Grafana :3000, Prometheus :9090, Alertmanager :9093, Loki :3100, cAdvisor :8081 (Docker Compose, `/opt/monitoring`). VulnNotes container renamed to `vnotes`. |
| 2026-09-19 | .99 | k3s workloads inventoried — nginx (:30453), juice-shop (:30300), vampi (:30500); node_exporter + Alloy (k3s pod logs) added. |
| 2026-09-19 | .100 | Jenkins pinned to `myjenkins-noname:2.568.3-jdk21`; cAdvisor + node_exporter + podman-exporter + Alloy added. |
| 2026-09-19 | .101 | crAPI stack healthy; juice-shop confirmed UP; node_exporter + podman-exporter + Alloy added. |
| 2026-09-19 | .102 | Added `noname-mcp` (:8013) and `vampi-mcp` container (:5000); node_exporter + podman-exporter + Alloy added. |
| 2026-09-18 | .98–.102 | Metrics monitoring installed (node_exporter, cAdvisor, podman-exporter, k3s kubelet scrape). |
| 2026-08-22 | .98 | Docker CE installed; VulnNotes API deployed; lab scripts at `~/notes-test/`. |
| 2026-08-22 | .100 | Docker CE installed alongside Podman; Jenkins stack deployed. |
