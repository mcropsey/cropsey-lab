# Lab Inventory — 192.168.1.98–102
**Updated:** 2026-08-22 | Rocky Linux 9.8 | Hyper-V VMs | All hosts verified via SSH

---

## Diagram

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                          192.168.1.x Lab Network                              │
│                                                                               │
│  .98 hv-rocky-linux-1            .99 hv-rocky-linux-2                        │
│  ┌───────────────────────────┐   ┌──────────────────────────────────┐         │
│  │ [Docker]                  │   │ k3s (Kubernetes v1.36.3+k3s1)    │         │
│  │  vulnnotes   :8000        │   │ cloudflared (Cloudflare Tunnel)  │         │
│  │ mcp-heartbeat svc         │   │ :80 :443 :6443 :8181 :8443       │         │
│  │ mcp-sweep svc             │   └──────────────────────────────────┘         │
│  └───────────────────────────┘                                                │
│                                                                               │
│  .100 hv-rocky-linux-3           .101 hv-rocky-linux-4                       │
│  ┌───────────────────────────┐   ┌──────────────────────────────────┐         │
│  │ [Podman rootful]          │   │ noname-sensor svc                │         │
│  │  kong:3.6   :80 :8001     │   │ [Podman rootful]                 │         │
│  │ [Docker]                  │   │  postgresdb    :5432 (internal)  │         │
│  │  jenkins    :8080 :50000  │   │  mongodb       :27017 (internal) │         │
│  │  jenkins-docker  :2376    │   │  chromadb      :8000 (internal)  │         │
│  └───────────────────────────┘   │  mailhog       :8025             │         │
│                                  │  gateway-svc   :443 (internal)   │         │
│                                  │  crapi-identity :8080/:8989       │         │
│                                  │  crapi-community :6060            │         │
│                                  │  crapi-workshop                  │         │
│                                  │  crapi-chatbot  :5500            │         │
│                                  │  crapi-web      :8888/:8443      │         │
│                                  │                 :30080/:30443    │         │
│                                  │  searxng        :8080            │         │
│                                  │  juice-shop     :3000            │         │
│                                  └──────────────────────────────────┘         │
│                                                                               │
│  .102 hv-rocky-linux-5                                                        │
│  ┌──────────────────────────────────────┐                                     │
│  │ noname-sensor svc                    │                                     │
│  │ crapi-mcp svc  :8009                 │                                     │
│  │ ai-sim svc     :8011 (LLM)           │                                     │
│  │                :8012 (GenAI)         │                                     │
│  └──────────────────────────────────────┘                                     │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## Host Summary

| IP | Hostname | Role | RAM | Disk Used | Uptime |
|----|----------|------|-----|-----------|--------|
| .98 | hv-rocky-linux-1 | VulnNotes API + MCP session host | 3.8 Gi | 7.9 G / 70 G (12%) | 2d 21h |
| .99 | hv-rocky-linux-2 | k3s + Cloudflare ingress | 5.5 Gi | 14 G / 70 G (20%) | 2d 21h |
| .100 | hv-rocky-linux-3 | Kong API Gateway + Jenkins CI | 7.8 Gi | 11 G / 70 G (15%) | 2d 21h |
| .101 | hv-rocky-linux-4 | crAPI lab + Noname Sensor | 7.4 Gi | 17 G / 70 G (23%) | 2d 21h |
| .102 | hv-rocky-linux-5 | Noname Sensor + MCP/AI sim | 3.1 Gi | 13 G / 70 G (19%) | 2d 21h |

Hardware (all VMs): Intel i9-9900K @ 3.60 GHz | Rocky Linux 9.8 (Blue Onyx) | Hyper-V

---

## .98 — hv-rocky-linux-1

**Role:** VulnNotes API lab target + MCP session host

### Systemd Services

| Service | Description | Status |
|---------|-------------|--------|
| `docker.service` | Docker CE container runtime | Running |
| `mcp-heartbeat.service` | MCP heartbeat — long-lived session for Noname | Running |
| `mcp-sweep.service` | MCP full sweep — long-lived session, reads | Running |
| `firewalld.service` | Firewall | Running |

### Docker Containers

| Name | Image | Status | Ports |
|------|-------|--------|-------|
| `vulnnotes` | `vulnnotes:latest` | Up (healthy) | `0.0.0.0:8000→8000` |

### Docker Volumes

| Volume | Purpose |
|--------|---------|
| `vulnnotes-data` | SQLite DB persistence for VulnNotes |

### Key URLs

| URL | What |
|-----|------|
| http://192.168.1.98:8000 | VulnNotes interactive SPA — login, note management, BOLA Lab |
| http://192.168.1.98:8000/docs | Swagger UI |
| http://192.168.1.98:8000/openapi.json | OpenAPI 3.1 spec (import into Active Testing) |
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
| `normal_traffic.py` | Generates legitimate baseline API traffic (4 concurrent users) | `python3 normal_traffic.py --base-url http://192.168.1.98:8000 --duration 120` |
| `exploit_bola.py` | Demonstrates BOLA — cross-user note read/write/delete | `python3 exploit_bola.py --base-url http://192.168.1.98:8000 --aggressive` |
| `requirements.txt` | `httpx>=0.27.0` | `pip3 install -r requirements.txt` |

### Runtime

| Item | Version |
|------|---------|
| Python | 3.9.25 |
| httpx | 0.28.1 |
| Docker CE | 29.x |

---

## .99 — hv-rocky-linux-2

**Role:** Kubernetes (k3s) cluster + Cloudflare Tunnel ingress

### Systemd Services

| Service | Description | Status |
|---------|-------------|--------|
| `k3s.service` | Lightweight Kubernetes v1.36.3+k3s1 | Running (control-plane, Ready) |
| `cloudflared.service` | Cloudflare Tunnel client | Running |

### Listening Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 80 | TCP | k3s ingress HTTP |
| 443 | TCP | k3s ingress HTTPS |
| 6443 | TCP | k3s API server |
| 8181 | TCP | k3s (Traefik / metrics) |
| 8443 | TCP | k3s ingress alt HTTPS |

### Notes

- Single-node cluster — `hv-rocky-linux-2` is both control-plane and worker
- Cloudflared routes external tunnel traffic into the k3s ingress

---

## .100 — hv-rocky-linux-3

**Role:** Kong API Gateway (Podman) + Jenkins CI/CD (Docker)

### Systemd Services

| Service | Description | Status |
|---------|-------------|--------|
| `docker.service` | Docker CE container runtime | Running |
| `firewalld.service` | Firewall | Running |

### Podman Containers (rootful)

| Name | Image | Status | Ports |
|------|-------|--------|-------|
| `kong` | `kong:3.6` | Up 2 days | `0.0.0.0:80→8000`, `0.0.0.0:8001→8001`, 8443-8444 |

### Docker Containers

| Name | Image | Status | Ports |
|------|-------|--------|-------|
| `jenkins` | `myjenkins-noname:latest` | Up (healthy) | `0.0.0.0:8080→8080`, `0.0.0.0:50000→50000` |
| `jenkins-docker` | `docker:dind` | Up | `0.0.0.0:2376→2376` |

### Docker Volumes

| Volume | Purpose |
|--------|---------|
| `jenkins-data` | Jenkins home — all jobs, config, credentials |
| `jenkins-docker-certs` | TLS certs for DinD communication |

### Key URLs

| URL | What |
|-----|------|
| http://192.168.1.100:8080 | Jenkins CI/CD web UI |
| http://192.168.1.100:8080/blue | Blue Ocean pipeline view |
| http://192.168.1.100:80 | Kong proxy (HTTP) |
| http://192.168.1.100:8001 | Kong Admin API |

### Jenkins Details

| Item | Value |
|------|-------|
| Admin user | `admin` |
| Image | `myjenkins-noname:latest` — Jenkins LTS + JDK17 + Docker CLI + 8 pre-installed plugins |
| Plugins pre-installed | blueocean, docker-workflow, github, github-branch-source, pipeline-model-definition, credentials-binding, plain-credentials, git, workflow-aggregator |
| GitHub credential ID | `github-token` (Classic PAT, Secret text) |
| DinD port | 2376 (TLS) |

### Docker Images on .100

| Repository | Tag | Size |
|------------|-----|------|
| `myjenkins-noname` | latest | 1.21 GB |
| `docker` | dind | 538 MB |

---

## .101 — hv-rocky-linux-4

**Role:** crAPI vulnerable app lab + Noname Sensor

### Systemd Services

| Service | Description | Status |
|---------|-------------|--------|
| `noname-sensor.service` | Noname Security API traffic sensor | Running |
| `firewalld.service` | Firewall | Running |

### Podman Containers (rootful)

| Name | Image | Status | Ports |
|------|-------|--------|-------|
| `postgresdb` | `postgres:14` | Up 2 days (healthy) | 5432 internal |
| `mongodb` | `mongo:4.4` | Up 2 days (healthy) | 27017 internal |
| `chromadb` | `chromadb/chroma:latest` | Up 2 days (healthy) | 8000 internal |
| `mailhog` | `crapi/mailhog:latest` | Up 2 days (healthy) | `0.0.0.0:8025→8025` |
| `api.mypremiumdealership.com` | `crapi/gateway-service:latest` | Up 2 days (healthy) | 443 internal |
| `crapi-identity` | `crapi/crapi-identity:latest` | Up 2 days (healthy) | 8080, 8989, 10001 internal |
| `crapi-community` | `crapi/crapi-community:latest` | Up 2 days (healthy) | 6060 internal |
| `crapi-workshop` | `crapi/crapi-workshop:latest` | Up 2 days (healthy) | — |
| `crapi-chatbot` | `crapi/crapi-chatbot:latest` | Up 2 days | `0.0.0.0:5500→5500` |
| `crapi-web` | `crapi/crapi-web:latest` | Up 2 days (healthy) | `:8888→80`, `:8443→443`, `:30080→80`, `:30443→443` |
| `searxng` | `searxng/searxng:latest` | Up 37 hours | `0.0.0.0:8080→8080` |
| `juice-shop` | `bkimminich/juice-shop` | Up | `0.0.0.0:3000→3000` |

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
| http://192.168.1.101:3000 | OWASP Juice Shop (⚠ DOWN) |

---

## .102 — hv-rocky-linux-5

**Role:** Noname Sensor + crAPI MCP server + AI simulator

### Systemd Services

| Service | Description | Status | Port |
|---------|-------------|--------|------|
| `noname-sensor.service` | Noname Security API traffic sensor | Running | — |
| `crapi-mcp.service` | crAPI MCP Server (Streamable HTTP) | Running | 8009 |
| `ai-sim.service` | AI-Sim — LLM + GenAI API simulator for sensor discovery | Running | 8011 (LLM), 8012 (GenAI) |
| `firewalld.service` | Firewall | Running | — |

### Key URLs

| URL | What |
|-----|------|
| http://192.168.1.102:8009 | crAPI MCP Server |
| http://192.168.1.102:8011 | AI-Sim LLM API |
| http://192.168.1.102:8012 | AI-Sim GenAI API |

---

## Change Log

| Date | Host | Change |
|------|------|--------|
| 2026-08-22 | .98 | Docker CE installed; VulnNotes API deployed (port 8000, `vulnnotes:latest`, SQLite volume `vulnnotes-data`) |
| 2026-08-22 | .98 | Lab scripts installed at `~/notes-test/` — `normal_traffic.py`, `exploit_bola.py`, `requirements.txt` |
| 2026-08-22 | .100 | Docker CE installed alongside existing rootful Podman; Kong continues running under Podman |
| 2026-08-22 | .100 | Jenkins CI/CD stack deployed — `jenkins` (myjenkins-noname:latest) on :8080/:50000, `jenkins-docker` (DinD) on :2376, volumes `jenkins-data` + `jenkins-docker-certs` |
| 2026-08-22 | .101 | juice-shop restarted — running on :3000 |
