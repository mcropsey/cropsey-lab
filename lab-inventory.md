# Lab Inventory — 192.168.1.98–102
**Updated:** 2026-08-21 | Rocky Linux 9.8 | Hyper-V VMs

---

## Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                       192.168.1.x Lab Network                            │
│                                                                          │
│  .98 hv-rocky-linux-1       .99 hv-rocky-linux-2                        │
│  ┌─────────────────────┐    ┌──────────────────────────────────┐         │
│  │ mcp-heartbeat svc   │    │ k3s (Kubernetes)                 │         │
│  │ mcp-sweep svc       │    │ cloudflared (Cloudflare Tunnel)  │         │
│  └─────────────────────┘    │ :80 :443 :6443 :8181 :8443       │         │
│                             └──────────────────────────────────┘         │
│  .100 hv-rocky-linux-3      .101 hv-rocky-linux-4                       │
│  ┌─────────────────────┐    ┌──────────────────────────────────┐         │
│  │ [rootful podman]    │    │ noname-sensor svc                │         │
│  │  kong:3.6           │    │ [rootful podman]                 │         │
│  │   :80→8000          │    │  postgresdb    :5432 (internal)  │         │
│  │   :8001 admin       │    │  mongodb       :27017 (internal) │         │
│  └─────────────────────┘    │  chromadb      :8000 (internal)  │         │
│                             │  mailhog       :8025             │         │
│                             │  gateway-svc   :443 (internal)   │         │
│                             │  crapi-identity :8080/:8989      │         │
│                             │  crapi-community :6060           │         │
│                             │  crapi-workshop                  │         │
│                             │  crapi-chatbot :5500             │         │
│                             │  crapi-web     :8888/:8443       │         │
│                             │                :30080/:30443     │         │
│                             │  juice-shop    :3000             │         │
│                             │  searxng       :8080             │         │
│                             └──────────────────────────────────┘         │
│  .102 hv-rocky-linux-5                                                   │
│  ┌──────────────────────────────────────┐                                │
│  │ noname-sensor svc                    │                                │
│  │ crapi-mcp svc  :8009                 │                                │
│  │ ai-sim svc     :8011 (LLM) :8012 (GenAI)                             │
│  └──────────────────────────────────────┘                                │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Hosts

| IP | Hostname | Role | Custom Services | Containers |
|---|---|---|---|---|
| .98 | hv-rocky-linux-1 | MCP session host | mcp-heartbeat, mcp-sweep, firewalld | None |
| .99 | hv-rocky-linux-2 | k3s + Cloudflare ingress | k3s, cloudflared | None (k3s-managed) |
| .100 | hv-rocky-linux-3 | Kong API Gateway | firewalld | kong:3.6 (rootful) |
| .101 | hv-rocky-linux-4 | crAPI lab + Noname Sensor | noname-sensor, firewalld | 12 (rootful, see below) |
| .102 | hv-rocky-linux-5 | Noname Sensor + MCP/AI sim | noname-sensor, crapi-mcp :8009, ai-sim :8011/:8012, firewalld | None |

---

## .100 Containers

| Name | Image | Status | Ports |
|---|---|---|---|
| `kong` | `kong:3.6` | Up (rootful) | :80→8000, :8001 admin |

---

## .101 Containers (all rootful)

| Name | Image | Status | Ports |
|---|---|---|---|
| `postgresdb` | `postgres:14` | Up (healthy) | 5432 internal |
| `mongodb` | `mongo:4.4` | Up (healthy) | 27017 internal |
| `chromadb` | `chromadb/chroma` | Up (healthy) | 8000 internal |
| `mailhog` | `crapi/mailhog` | Up (healthy) | :8025 |
| `api.mypremiumdealership.com` | `crapi/gateway-service` | Up (healthy) | 443 internal |
| `crapi-identity` | `crapi/crapi-identity` | Up (healthy) | 8080, 8989, 10001 internal |
| `crapi-community` | `crapi/crapi-community` | Up (healthy) | 6060 internal |
| `crapi-workshop` | `crapi/crapi-workshop` | Up (healthy) | — |
| `crapi-chatbot` | `crapi/crapi-chatbot` | Up | :5500 |
| `crapi-web` | `crapi/crapi-web` | Up (healthy) | :8888/:8443, :30080/:30443 |
| `juice-shop` | `bkimminich/juice-shop` | Up | :3000 |
| `searxng` | `searxng/searxng` | Up | :8080 |

---

## Change Log

| Date | Change |
|---|---|
| 2026-08-21 | juice-shop restarted — crashed via NoSQL injection (`$where` eval); monitor for repeats |
| 2026-08-21 | Rootless kong container removed from .100 |
