# Lab Inventory — 192.168.1.x home lab
**Updated:** 2026-09-29 | 13 hosts verified live via SSH | Rocky Linux 9.8 / 10.2 Hyper-V VMs + 5x Raspberry Pi 5 (Ubuntu 24.04)
New since 2026-09-19: **LiteLLM AI gateway on .101**, the **Agent Lab** (6 scheduled AI agents on .100/.85/.75/.76),
full monitoring coverage of rk1–rk4 and .103–.105, and sections for the microk8s cluster and the RHCSA practice lab.

### Where the documentation lives

| Doc | Location | Use it for |
|---|---|---|
| Lab inventory (this file) | `~/cropsey-lab/lab-inventory.md` on .101 | every host, service, port and change |
| Topology diagram | `~/cropsey-lab/home-lab-topology.html` → `home-lab-topology.png` (light), `cropsey-lab.png` (dark) | visual overview; PNGs are rendered from the HTML (see below) |
| Agent Lab overview | `~/agent-lab/README.md` on .101 | what the AI agents do, safeguards, day-to-day commands |
| Agent Lab manual rebuild | `~/agent-lab/docs/MANUAL-SETUP.md` | rebuild/verify/tear down by hand, no scripts or AI |
| Agent Lab build log | `~/agent-lab/docs/BUILD-NOTES.md` | timeline, decisions, incidents, every change made, follow-ups |
| Monitoring runbook | `~/MONITORING_SETUP.md` on rasp5 (.85) | Prometheus/Grafana/Loki/Alloy install |
| Previous diagrams | `~/cropsey-lab/archive/2026-09-19/` | the 2026-09-19 PNGs |
| GitHub | `github.com/mcropsey/cropsey-lab` (**public**: these docs) · `github.com/mcropsey/agent-lab` (**private**: Agent Lab code + docs) | off-box copies; .101 pushes with repo-specific deploy keys (`~/.ssh/github_cropsey_lab`, `~/.ssh/github_agent_lab`, SSH aliases `github-cropsey-lab` / `github-agent-lab` in `~/.ssh/config`) |

Re-render the PNGs after editing the HTML (on .101, needs `chromium-headless`):
```bash
cd ~/cropsey-lab; HS=/usr/lib64/chromium-browser/headless_shell; URL="file://$PWD/home-lab-topology.html"
for t in light dark; do
  H=$($HS --no-sandbox --window-size=1280,900 --virtual-time-budget=15000 --dump-dom "$URL?measure&theme=$t" 2>/dev/null | grep -o 'H=[0-9]*' | cut -d= -f2)
  out=$([ $t = light ] && echo home-lab-topology.png || echo cropsey-lab.png)
  $HS --no-sandbox --hide-scrollbars --force-device-scale-factor=2 --window-size=1280,$H --virtual-time-budget=15000 --screenshot=$PWD/$out "$URL?theme=$t"
done
```

---

## Network & Service Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                                  192.168.1.x Home Lab                                          │
│                                                                                                │
│  .85 rasp5 (RPi 5)            CONTROL NODE — Claude Code/opencode, inventory, runbooks          │
│    [Docker] twingate-jade-limpet · agentlab (analyst, news) · node_exporter · alloy             │
│                                                                                                │
│  .98  hv-rocky-linux-1  ★ OBSERVABILITY HUB   grafana:3000 prometheus:9090 alertmanager:9093    │
│                                               loki:3100 cadvisor:8081 · vnotes:8000 (VulnNotes) │
│  .99  hv-rocky-linux-2  k3s v1.36 + Cloudflare Tunnel — juice-shop:30300 vampi:30500 nginx      │
│  .100 hv-rocky-linux-3  Kong:80/8001 (Podman) · Jenkins:8080 (Docker) · agentlab (triage, sre)  │
│  .101 hv-rocky-linux-4  ★ LiteLLM AI GATEWAY :4000 (Podman) · crAPI stack · juice-shop · searxng│
│  .102 hv-rocky-linux-5  Noname sensor · crapi-mcp:8009 noname-mcp:8013 ai-sim:8011/8012 vampi-mcp│
│                                                                                                │
│  .75 rk1 ┐ microk8s v1.29 (2 nodes): vampi x2, ingress   · agentlab (conversation)              │
│  .76 rk2 ┘                                                · agentlab (reviewer)                  │
│  .77 rk3 · .78 rk4    spare Raspberry Pi 5s (monitoring agents only; reserved)                  │
│                                                                                                │
│  .103 hv-rocky-linux-6 ┐ RHCSA RHEL10 practice lab (Rocky 10.2, GNOME + Cockpit :9090)         │
│  .104 hv-rocky-linux-7 ┘                                                                        │
│  .105 utility.lab      dnf repo mirror (httpd :80) + NFS exports                                │
│                                                                                                │
│  .194 (not managed here)  LM Studio on RTX 5090 :1234 — qwen3.8-27b, backend for LiteLLM        │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## AI Gateway & Agent Lab (new 2026-09-29)

```
 AGENTS (one scheduler per host)            GATEWAY (.101)                        MODEL (.194)
 .100 hv-rocky-linux-3  triage, sre, chaos ─┐                                     ┌───────────────────┐
 .85  rasp5             analyst, news ──────┤  one virtual key   ┌──────────────┐  │ LM Studio :1234   │
 .75  rk1               conversation ───────┼──────per persona──►│ LiteLLM :4000├─►│ qwen3.8-27b       │
 .76  rk2               reviewer ───────────┘  (team agent-lab)  │ route        │  │ RTX 5090 (~78 t/s)│
                                                                 │ agentlab-main│  └───────────────────┘
      │ 1 JSON line per run                                      │ spend logs → Postgres (postgresdb)
      └──► docker / journald ──► Alloy ──► Loki .98 ──► Grafana "Agent Lab" dashboard
```

### How it works

- Each agent host runs one **scheduler** (APScheduler) for the personas assigned to it in `~/agent-lab/hosts.conf`.
  Docker hosts (.100, .85) run it as container `agentlab` (Docker Compose); rk1/rk2 run it as `agentlab.service`
  (Python venv + systemd).
- Every wake-up re-reads `config/lab.yaml`, picks a jittered wait from the persona's rate × global intensity ×
  time of day, and runs the persona. **08:00–22:00 (America/Chicago) is the daytime trickle** (~1.4 runs/hour
  lab-wide) because the 5090 is used interactively; 22:00–08:00 runs ~10.5 runs/hour.
- Agents call LiteLLM with the OpenAI-compatible API (`OPENAI_BASE_URL=http://192.168.1.101:4000/v1`), each
  persona with its **own virtual key**. Requests are tagged `agent-lab`, `persona:<name>`, `host:<hostname>`.
- Each persona is also **registered in LiteLLM's agent registry** (UI → *Agents*): 7 agents, `agent-triage` …
  `agent-reviewer`, each with an agent card (description, framework, host, skills). Its virtual key carries the
  matching `agent_id`, so every request is attributed to the agent and the Agents page shows per-agent spend and
  activity. Chaos isn't registered. The cards have no A2A URL: the agents are scheduled clients, not A2A servers,
  so they can't be invoked from the UI.
- A shared HTTP layer meters every call (tokens, tool calls, latency, LiteLLM's cost header) and ~2% of
  calls are sent deliberately wrong (bad model name → 403, over-long prompt → 400). LiteLLM rejects both
  before they reach LM Studio.
- Each run prints **one JSON log line** (persona, host, model, tokens, latency, tool calls, success, errors,
  cost, persona-specific fields). Alloy ships it to Loki; the Grafana dashboard is built from those lines.

### Personas

| Persona | Framework | Host | Key(s) | What it does |
|---|---|---|---|---|
| triage | PydanticAI | .100 | agent-triage | Synthetic support tickets (Faker, some fake PII, half written by the LLM). Tools: customer lookup, status page. Outputs category, priority, PII flags, draft reply. |
| sre | LangGraph | .100 | agent-sre | Generates 300–1500 lines of service logs with one incident (DB pool, OOM, expired cert, disk full, latency, login failures), investigates with list/read/search tools, writes an incident summary. |
| analyst | LangGraph | .85 | agent-analyst | Random business questions over the Chinook SQLite DB with a read-only SQL tool; retries failed queries (20% of runs get a stale schema hint). |
| news | PydanticAI | .85 | agent-news | Pulls 2–3 public RSS feeds, summarizes and tags topics; canned articles if offline. |
| conversation | OpenAI Agents SDK | .75 rk1 | agent-customer, agent-support | Customer agent (random persona/goal) and support agent (fake CRM tools) talk for 3–8 turns. |
| reviewer | OpenAI Agents SDK | .76 rk2 | agent-reviewer | Reviews bundled diffs with planted bugs or synthetic diffs of the lab's own code; long-context (up to ~15k tokens). |
| chaos | httpx | .100 | agent-chaos | ~6 rate-limit bursts/day: 5 simultaneous calls on a 1-in-flight / 2-rpm key → LiteLLM 429s. |

Red-team persona deferred; key `agent-redteam` exists but is unused.

### Safeguards for the LM Studio GPU

| Guard | Where | Effect |
|---|---|---|
| Model lock | agent keys allow only `agentlab-main` / `agentlab-main-patient` | No agent can make LM Studio load another model (403). |
| 1 request in flight | `max_parallel_requests: 1` on the `agentlab-main` route | LiteLLM queues agent calls; verified 3 concurrent → served one after another. |
| Daytime trickle | `config/lab.yaml` schedule | 13% of night rate 08:00–22:00. |
| Back-off | scheduler | A slow daytime call (>20 s at <20 tok/s) pauses that host 30 min. |
| Per-key limits | LiteLLM | rpm 20–30, tpm 120k–250k, $1/day simulated budget per key. |
| Pre-call context check | `router_settings.enable_pre_call_checks` | Over-long prompts rejected by LiteLLM, not the GPU. |

### Operating it (from `~/agent-lab` on .101)

| Task | Command |
|---|---|
| Status of every host | `./deploy.sh status` |
| Run one persona now | `./deploy.sh run <persona> [--no-errors]` |
| Change volume | edit `global.intensity` in `config/lab.yaml` → `./deploy.sh config` |
| Kill switch (pause) | `global.enabled: false` → `./deploy.sh config` |
| Stop everything | `./stop.sh` (one host: `./stop.sh 192.168.1.100`) |
| Redeploy / start | `./deploy.sh` (one host: `./deploy.sh 192.168.1.85`) |
| Dashboard | http://192.168.1.98:3000/d/agent-lab (folder **Agent Lab**) |
| Spend / requests | LiteLLM UI → Logs / Usage (key alias `agent-*`, tag `persona:*`) |
| Per-agent view | LiteLLM UI → **Agents** (7 registered agents, spend per agent) |

### Registered agents (LiteLLM UI → Agents)

| Agent | Card name | Framework | Host | Linked key |
|---|---|---|---|---|
| agent-triage | Ticket triage | PydanticAI | .100 | agent-triage |
| agent-sre | SRE log analyst | LangGraph | .100 | agent-sre |
| agent-analyst | Data analyst | LangGraph | .85 | agent-analyst |
| agent-news | News digest | PydanticAI | .85 | agent-news |
| agent-customer | Simulated customer | OpenAI Agents SDK | .75 rk1 | agent-customer |
| agent-support | Customer support | OpenAI Agents SDK | .75 rk1 | agent-support |
| agent-reviewer | Code reviewer | OpenAI Agents SDK | .76 rk2 | agent-reviewer |

Registered and linked by `~/agent-lab/scripts/setup_keys.sh` (idempotent, matched by agent name). Spend is
attributed from 2026-09-29 21:25 CDT on; earlier test traffic isn't linked to an agent.

Repo: `~/agent-lab` on .101 (git). Details in its README; manual rebuild in `docs/MANUAL-SETUP.md`; build log in `docs/BUILD-NOTES.md`.

---

## Observability / Monitoring Topology

Installed 2026-09-18 (metrics) and 2026-09-19 (logs); coverage extended to all 13 hosts since. Full runbook:
`~/MONITORING_SETUP.md` on rasp5; helper scripts in `~/monitoring/`.

```
                         ┌──────────────────────────────────────────┐
                         │  .98  /opt/monitoring  (Docker Compose,   │
   metrics  ──scrape──►  │  Prometheus :9090 ──► Alertmanager :9093  │
   (pull)                │        ▲ remote-write (log-derived series)│
   logs     ──push────►  │  Loki :3100  ──ruler──► Alertmanager      │
   (Alloy)               │  Grafana :3000 ◄── queries both           │
                         │  cAdvisor :8081 (local container metrics) │
                         └──────────────────────────────────────────┘
  node_exporter :9100 → all 13 hosts (.85 .98–.105 rk1–rk4)      Alloy → Loki: all 13 hosts
  cAdvisor      :8081 → .98 .100                                  (journal + auditd; docker on .98 .100 .85;
  podman-exp    :9882 → .100 .101 .102 .104                        k3s pods on .99)
  k3s kubelet   :10250 → .99 (SA-token, TLS-verified)
```
Prometheus: 24 active targets, all **up** (2026-09-29).

**Notes**
- Grafana dashboard folders on .98: Fleet Overview, k3s Cluster (.99), Gateway + CI Host (.100), crAPI Stack (.101),
  MCP Host (.102), Logs, **Raspberry Pi Cluster**, **Agent Lab**.
- ⚠ `gen_dashboards.py` (rasp5) does **not** generate the *Raspberry Pi Cluster* or *Agent Lab* folders, and its
  documented deploy step runs `rm -rf /opt/monitoring/grafana/dashboards/*` — running it as written deletes both.
  The Agent Lab dashboard has its own generator/installer: `~/agent-lab/grafana/install.sh` (touches only its folder).
- Prometheus config is a mounted **directory** (`prometheus/config/`); reload with `curl -XPOST localhost:9090/-/reload`.
- Alertmanager receiver is still `null` — alerts fire but are **not delivered anywhere**.
- The Grafana admin password in `/opt/monitoring/.env` no longer works (changed in the UI since install).
- Firewalls: .98/.101/.102 use firewalld rich rules (**never** `firewall-cmd --reload` there);
  .99/.100 use nft table `inet monitoring_acl` via `monitoring-acl.service`.

---

## Host Summary

| IP | Hostname | Role | OS | RAM (total / avail) | Disk used | Uptime |
|----|----------|------|----|-----|-----------|--------|
| .85 | rasp5 | Control node, Twingate, **agents: analyst + news** | Ubuntu 24.04.5 (RPi 5) | 7.7 / 5.2 Gi | 16 / 117 G | 16h |
| .98 | hv-rocky-linux-1 | **Observability hub** + VulnNotes API + MCP session host | Rocky 9.8 | 7.5 / 4.9 Gi | 19 / 70 G | 4d |
| .99 | hv-rocky-linux-2 | k3s cluster + Cloudflare ingress | Rocky 9.8 | 3.6 / 1.5 Gi | 15 / 70 G | 4d |
| .100 | hv-rocky-linux-3 | Kong + Jenkins CI, **agents: triage + sre + chaos** | Rocky 9.8 | 7.5 / 4.9 Gi | 15 / 70 G | 12h |
| .101 | hv-rocky-linux-4 | **LiteLLM AI gateway** + crAPI lab + Noname sensor | Rocky 9.8 | 3.6 / 1.2 Gi | 19 / 70 G | 4d |
| .102 | hv-rocky-linux-5 | Noname sensor + MCP servers + AI sim | Rocky 9.8 | 3.6 / 2.1 Gi | 19 / 70 G | 4d |
| .103 | hv-rocky-linux-6 | RHCSA practice workstation | Rocky 10.2 | 3.6 / 2.6 Gi | 6.5 / 70 G | 4d |
| .104 | hv-rocky-linux-7 | RHCSA practice workstation | Rocky 10.2 | 3.6 / 2.5 Gi | 11 / 70 G | 4d |
| .105 | utility.lab.example.com | RHCSA utility server: dnf repos + NFS | Rocky 10.2 | 3.6 / 2.6 Gi | 17 / 70 G | 4d |
| .75 | rk1 | microk8s node, **agent: conversation** | Ubuntu 24.04.4 (RPi 5) | 7.8 / 6.5 Gi | 8.6 / 58 G | 4d |
| .76 | rk2 | microk8s node, **agent: reviewer** | Ubuntu 24.04.4 (RPi 5) | 7.8 / 6.4 Gi | 8.7 / 58 G | 4d |
| .77 | rk3 | spare (reserved) | Ubuntu 24.04.4 (RPi 5) | 7.8 / 7.3 Gi | 3.9 / 58 G | 2d |
| .78 | rk4 | spare (reserved) | Ubuntu 24.04.4 (RPi 5) | 7.8 / 7.3 Gi | 4.0 / 58 G | 4d |

Rocky VMs: Hyper-V on an Intel i9-9900K @ 3.60 GHz. Raspberry Pis: Pi 5 Model B, aarch64.
⚠ RAM on .99 (was 6.0 Gi) and .101 (was 8.1 Gi) now reports 3.6 Gi — cause not verified (Hyper-V dynamic
memory or a VM resize). .101 is tight with LiteLLM added; keep new workloads off it.

---

## .85 — rasp5

**Role:** Control / management node — Claude Code and opencode run here; holds the monitoring runbook
(`~/MONITORING_SETUP.md`) and staging (`~/monitoring/`). Also runs two Agent Lab personas.

| Service / container | Description | Status |
|---------|-------------|--------|
| `twingate-jade-limpet` (Docker) | Twingate connector (remote access) | Up (healthy) |
| `agentlab` (Docker Compose, `~/agent-lab`) | Agent Lab scheduler — personas **analyst**, **news** | Up |
| `node_exporter.service` | :9100, scraped by Prometheus on .98 | Running |
| `alloy.service` | journal + docker logs → Loki | Running |
| `glances.service` / `atop` / `cups` | local monitoring, accounting, printing | Running |

- Docker Compose CLI plugin v5.5.1 installed per-user (`~/.docker/cli-plugins`) for the Agent Lab (2026-09-29).
- SSH mesh hub: passwordless to .98–.102 and rk1–rk4 with `~/.ssh/id_ed25519`; **.103–.105 still reject this key**
  (they accept the key on .101).

---

## .98 — hv-rocky-linux-1

**Role:** Observability hub (Grafana/Prometheus/Loki/Alertmanager) + VulnNotes API lab target + MCP session host

| Container (Docker) | Image | Ports |
|------|-------|-------|
| `grafana` | `grafana/grafana:13.2.2` | :3000 (host net) |
| `prometheus` | `prom/prometheus:v3.14.0` | :9090 (host net) |
| `alertmanager` | `prom/alertmanager:v0.34.1` | :9093 (host net) |
| `loki` | `grafana/loki:3.7.8` | :3100 (host net) |
| `cadvisor` | `ghcr.io/google/cadvisor:v0.60.6` | :8081 (host net) |
| `vnotes` | `vnotes:latest` | 0.0.0.0:8000→8000 |
| `vulnnotes` | `vulnnotes:latest` | **Exited 13 days ago** (old copy of the VulnNotes app; safe to remove) |

Systemd: `docker`, `containerd`, `node_exporter`, `alloy`, `mcp-heartbeat`, `mcp-sweep`.

| URL | What |
|-----|------|
| http://192.168.1.98:3000 | Grafana (dashboards incl. **Agent Lab**: `/d/agent-lab`) |
| http://192.168.1.98:9090 | Prometheus |
| http://192.168.1.98:9093 | Alertmanager |
| http://192.168.1.98:3100 | Loki API (query via Grafana) |
| http://192.168.1.98:8081 | cAdvisor |
| http://192.168.1.98:8000 | VulnNotes SPA · `/docs` Swagger · `/openapi.json` · `/health` |

VulnNotes demo users: alice/alice123, bob/bob12345, charlie/charlie1 (user), admin/admin123 (admin).
Lab scripts in `~/notes-test/`: `normal_traffic.py` (baseline traffic), `exploit_bola.py` (BOLA demo).

---

## .99 — hv-rocky-linux-2

**Role:** k3s v1.36.3+k3s1 single-node cluster + Cloudflare Tunnel ingress (unchanged since 09-19)

| Namespace / workload | Service | External |
|------------------------|---------|----------|
| `default/nginx` | NodePort | :30453 |
| `juiceshop/juice-shop` | NodePort | :30300 |
| `vampi/vampi` (noname-sensor sidecar) | NodePort | :30500 |
| `ingress-nginx` controller | — | :80/:443/:8443 |
| kube-system: coredns, metrics-server, local-path-provisioner | ClusterIP | — |

- `cloudflared.service` routes tunnel traffic into ingress-nginx. k3s API :6443; kubelet :10250 scraped by Prometheus.
- ⚠ `sudo` drops `/usr/local/bin` — use `sudo /usr/local/bin/k3s kubectl ...`. Firewall: nft `inet monitoring_acl`.

---

## .100 — hv-rocky-linux-3

**Role:** Kong API Gateway (Podman) + Jenkins CI/CD (Docker) + Agent Lab personas triage/sre/chaos

| Container | Runtime | Image | Ports |
|------|---------|-------|-------|
| `kong` | Podman (rootful) | `kong:3.6` | 0.0.0.0:80→8000, :8001 admin |
| `jenkins` | Docker | `myjenkins-noname:2.568.3-jdk21` | :8080, :50000 |
| `jenkins-docker` | Docker | `docker:dind` | :2376 (TLS) |
| `cadvisor` | Docker | `ghcr.io/google/cadvisor:v0.60.6` | :8081 |
| `agentlab` | Docker Compose (`~/agent-lab`) | `agentlab:latest` (built locally) | — (outbound to .101:4000) |

Systemd: `docker`, `containerd`, `node_exporter`, `prometheus-podman-exporter` (:9882), `alloy`.
URLs: Jenkins http://192.168.1.100:8080 (`/blue`), Kong proxy :80, Kong admin :8001, cAdvisor :8081.
Rebooted ~2026-09-29 09:00 (all containers came back on their own). Firewall: nft `inet monitoring_acl`.

---

## .101 — hv-rocky-linux-4

**Role:** **LiteLLM AI gateway** + crAPI vulnerable-app lab + Noname Sensor. Agent Lab control repo lives here (`~/agent-lab`).

### LiteLLM (added 2026-09-29)

| Item | Value |
|------|-------|
| Container | `litellm` — `ghcr.io/berriai/litellm:main-stable`, rootful Podman, `0.0.0.0:4000→4000` |
| Config | `/opt/litellm/config.yaml` (bind-mounted read-only; edit then `sudo podman restart litellm`). Backups `config.yaml.bak*` alongside |
| Secrets | master key / DB URL / UI login in the container env (`/opt/litellm/litellm.env`, root-only) |
| Database | Postgres container `postgresdb` (shared with crAPI), database `litellm` — virtual keys, teams, spend logs |
| Networks | `docker_default` (10.89.0.0/24, with the crAPI stack) + `podman` |
| Backend | LM Studio at `http://192.168.1.194:1234/v1` (RTX 5090) |
| UI | http://192.168.1.101:4000/ui |

| Model name | Backend model | Notes |
|---|---|---|
| `qwen3.8-27b` | qwen/qwen3.8-27b | **Default model kept loaded in LM Studio** (~78 tok/s on the 5090) |
| `qwen3.6-35b-a3b`, `gemma-4-31b-qat`, `muse-glimmer` | — | Available, but calling them makes LM Studio load them next to qwen3.8 and pushes it partly onto CPU (~0.3 tok/s). Avoid unless you unload qwen3.8 first. |
| `nomic-embed-text` | text-embedding-nomic-embed-text-v1.5 | Embeddings |
| `agentlab-main` | qwen/qwen3.8-27b | Agent Lab route: 180 s timeout, 1 request in flight, simulated pricing, 32k input limit |
| `agentlab-main-patient` | qwen/qwen3.8-27b | Fallback for `agentlab-main` (600 s timeout); same model, so no model load |

`router_settings`: `enable_pre_call_checks: true`, fallback `agentlab-main → agentlab-main-patient`.
Agents page: 7 registered agents (`agent-triage` … `agent-reviewer`), each linked to its key via `agent_id`,
so spend/activity is attributed per agent (no A2A endpoint; not invokable from the UI).
Team **agent-lab** with 9 virtual keys (`agent-triage`, `-sre`, `-analyst`, `-news`, `-customer`, `-support`,
`-reviewer`, `-redteam` (unused), `-chaos`). Managed by `~/agent-lab/scripts/setup_litellm.sh` and `setup_keys.sh`
(idempotent; the agent-lab config blocks are marked and removable).

### Podman containers (rootful)

| Name | Image | Ports |
|------|-------|-------|
| `litellm` | `ghcr.io/berriai/litellm:main-stable` | :4000 |
| `postgresdb` | `postgres:14` | 5432 internal (crAPI + LiteLLM) |
| `mongodb` | `mongo:4.4` | 27017 internal |
| `chromadb` | `chromadb/chroma:latest` | 8000 internal |
| `mailhog` | `crapi/mailhog:latest` | :8025 |
| `api.mypremiumdealership.com` | `crapi/gateway-service:latest` | 443 internal |
| `crapi-identity` / `crapi-community` / `crapi-workshop` | `crapi/*` | internal |
| `crapi-chatbot` | `crapi/crapi-chatbot:latest` | :5500 |
| `crapi-web` | `crapi/crapi-web:latest` | :8888, :8443, :30080, :30443 |
| `searxng` | `searxng/searxng:latest` | :8080 |
| `juice-shop` | `bkimminich/juice-shop:latest` | :3000 |

Systemd: `noname-sensor` (eBPF), `node_exporter`, `prometheus-podman-exporter`, `alloy`.
URLs: crAPI http://192.168.1.101:8888 / https://…:8443, MailHog :8025, chatbot :5500, SearXNG :8080, Juice Shop :3000,
**LiteLLM :4000**. Firewall: firewalld rich rules — **never** `firewall-cmd --reload`.
Also installed 2026-09-29: `chromium-headless` (EPEL) for rendering these docs to PNG.

---

## .102 — hv-rocky-linux-5

**Role:** Noname Sensor + MCP servers + AI simulator (unchanged since 09-19)

| Service | Port |
|---------|------|
| `crapi-mcp.service` (crAPI MCP, Streamable HTTP) | 8009 |
| `noname-mcp.service` (Noname Management API MCP) | 8013 |
| `ai-sim.service` (LLM + GenAI API simulator) | 8011 / 8012 |
| `vampi-mcp` (Podman container) | 5000 |
| `noname-sensor`, `node_exporter`, `prometheus-podman-exporter`, `alloy` | — / 9100 / 9882 |

Services live under `/opt/{crapi-mcp,noname-mcp,ai-sim}` with their own `.venv`. firewalld rich rules — never `--reload`.

---

## .75 / .76 — rk1, rk2 (Raspberry Pi 5)

**Role:** 2-node **microk8s v1.29.15** cluster + Agent Lab personas (rk1: conversation, rk2: reviewer)

| microk8s workload | Status |
|---|---|
| `vampi/vampi` (2 pods) | Running |
| `ingress/nginx-ingress-microk8s-controller` (per node) | Running |
| kube-system: calico, coredns, hostpath-provisioner, metrics-server | Running |

Add-ons: dns, ha-cluster, helm/helm3, hostpath-storage, ingress, metrics-server, storage.
Agent Lab runs as `agentlab.service` (venv `~/agent-lab/.venv`, `python3-venv` installed 2026-09-29), logs to journald.
`node_exporter` + `alloy` running (log shipping TODO from 09-19 is done).

## .77 / .78 — rk3, rk4 (Raspberry Pi 5)

**Role:** spare — reserved for other plans. Only `node_exporter` and `alloy` run. rk3 rebooted ~2026-09-27.

---

## .103 / .104 / .105 — RHCSA RHEL10 practice lab (Rocky 10.2)

| Host | Role | Notable |
|---|---|---|
| .103 hv-rocky-linux-6 | practice workstation | GNOME (gdm), Cockpit :9090, CUPS |
| .104 hv-rocky-linux-7 | practice workstation | GNOME, Cockpit :9090, `prometheus-podman-exporter` :9882 |
| .105 utility.lab.example.com | utility server | httpd :80 serving dnf repos (`/rhel10/` BaseOS + AppStream, flatpak); NFS exports `/home/guests`, `/exports/docs`, `/exports/data` to 192.168.1.0/24 |

All three: `node_exporter` + `alloy` running and scraped/shipped to .98. No containers.

---

## Change Log

| Date | Host | Change |
|------|------|--------|
| 2026-09-29 | .101 | **LiteLLM documented** (not in the 09-19 doc; container last started 2026-09-29 14:38): :4000, Postgres-backed, LM Studio backend on .194. Added `agentlab-main` + `agentlab-main-patient` routes, fallback and pre-call checks (additive, backed up). Team `agent-lab` + 9 virtual keys. `chromium-headless` installed. |
| 2026-09-29 | .100 .85 .75 .76 | **Agent Lab deployed**: `agentlab` container on .100 (triage, sre, chaos) and .85 (analyst, news); `agentlab.service` on rk1 (conversation) and rk2 (reviewer). Docker Compose plugin added per-user on .85; `python3-venv` on rk1/rk2. |
| 2026-09-29 | .101 | Agent Lab documentation added: `~/agent-lab/docs/MANUAL-SETUP.md` (manual rebuild) and `BUILD-NOTES.md` (build log). |
| 2026-09-29 | .101 | 7 personas registered on LiteLLM's **Agents** page; each key linked via `agent_id` for per-agent spend. |
| 2026-09-29 | .98 | Grafana folder **Agent Lab** (`/d/agent-lab`) added. Noted: *Raspberry Pi Cluster* folder isn't produced by `gen_dashboards.py`. |
| 2026-09-29 | all | Monitoring now covers all 13 hosts (node_exporter + Alloy); podman-exporter also on .104. rk1–rk4 and .103–.105 documented. |
| 2026-09-29 | .99 .101 | Reported RAM down to 3.6 Gi (was 6.0 / 8.1 Gi); cause not verified. |
| 2026-09-19 | all | Full live re-verification via SSH; doc scope expanded to include rasp5 (.85). |
| 2026-09-19 | .85 | node_exporter v1.12.1, Alloy, glances, atop added; Twingate connector. |
| 2026-09-19 | .98 | Observability hub deployed (Grafana, Prometheus, Alertmanager, Loki, cAdvisor under `/opt/monitoring`). |
| 2026-09-19 | .99–.102 | Workloads inventoried; node_exporter, podman-exporter and Alloy added. |
| 2026-09-18 | .98–.102 | Metrics monitoring installed. |
| 2026-08-22 | .98 / .100 | Docker CE installed; VulnNotes API (.98) and Jenkins stack (.100) deployed. |
