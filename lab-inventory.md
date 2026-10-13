# Lab Inventory — 192.168.1.x home lab
**Updated:** 2026-10-07 | 13 hosts verified live via SSH | Rocky Linux 9.8 / 10.2 Hyper-V VMs + 5x Raspberry Pi 5 (Ubuntu 24.04)
New since 2026-09-29: **mikes-agents** (5 scheduled ops agents on .98: secnews, labwatch, cvewatch, patchwatch +
labinventory) using the local LLM through LiteLLM; the **Agent Lab** traffic generator was retired (~2026-10-05);
**security patching of all 11 managed hosts** on 2026-10-07 (reboots 22:05 the same night).

### Where the documentation lives

| Doc | Location | Use it for |
|---|---|---|
| Lab inventory (this file) | `~/cropsey-lab/lab-inventory.md` (git clone on .98) | every host, service, port and change |
| Topology diagram | `~/cropsey-lab/home-lab-topology.html` → `home-lab-topology.png` (light), `cropsey-lab.png` (dark) | visual overview; PNGs are rendered from the HTML (see below) |
| **Service map (auto, daily)** | [`service-map/`](service-map/) in this repo · live page `http://192.168.1.98:9440/` | every service, timer, cron job, container, k8s workload and AI agent per host, with dependencies; regenerated 05:40 by servicemap-agent and committed here when it changes |
| **Agent reports (all agents)** | `http://192.168.1.98:9440/reports/` (LAN only) · Grafana *Agent Lab · Daily Reports* | every labwatch/cvewatch/patchwatch/secnews/servicemap report by date with its headline; linked from the cropseyit.com footer on rasp5 |
| Lab agents (mikes-agents) | `~/mikes-agents/` on .98 (git) · per-agent READMEs | what each ops agent does, install, operate, troubleshoot |
| Patching log | `~/patching/2026-10-07/` on .98 | per-host dnf/apt output, `reboots.log` from the reboot run |
| Agent Lab (retired) | `~/agent-lab/` on .101 (README, `docs/MANUAL-SETUP.md`, `docs/BUILD-NOTES.md`) | history of the retired traffic generator |
| Monitoring runbook | `~/MONITORING_SETUP.md` on rasp5 (.85) | Prometheus/Grafana/Loki/Alloy install |
| Previous diagrams | `~/cropsey-lab/archive/2026-09-19/` | the 2026-09-19 PNGs |
| GitHub | `github.com/mcropsey/cropsey-lab` (these docs) · `mikes-agents` (lab agents) · `agent-lab` (retired) · `vnotes` — all **public** | Clones on .98 push over HTTPS with the token in `~/.git-credentials` (repo-local `credential.helper=store`). The old per-repo deploy keys on .101 are no longer accepted by GitHub. No keys or passwords in any repo. |

Re-render the PNGs after editing the HTML. `chromium-headless` is only on .101, so copy the HTML there, render,
and copy the PNGs back to the clone on .98:
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
│    [Docker] twingate-jade-limpet · node_exporter · alloy                                        │
│                                                                                                │
│  .98  hv-rocky-linux-1  ★ OBSERVABILITY HUB   grafana:3000 prometheus:9090 alertmanager:9093    │
│                                               loki:3100 cadvisor:8081 · vnotes:8000 (VulnNotes) │
│                         ★ LAB AGENTS (podman + timers): secnews · labwatch · cvewatch · patchwatch│
│  .99  hv-rocky-linux-2  k3s v1.36 + Cloudflare Tunnel — juice-shop:30300 vampi:30500 nginx      │
│  .100 hv-rocky-linux-3  Kong:80/8001 (Podman) · Jenkins:8080 (Docker)                           │
│  .101 hv-rocky-linux-4  ★ LiteLLM AI GATEWAY :4000 (Podman) · crAPI stack · juice-shop · searxng│
│  .102 hv-rocky-linux-5  Noname sensor · crapi-mcp:8009 noname-mcp:8013 ai-sim:8011/8012 vampi-mcp│
│                                                                                                │
│  .75 rk1 ┐ microk8s v1.29 (2 nodes): vampi x2, ingress                                          │
│  .76 rk2 ┘                                                                                      │
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

## AI Gateway & Lab Agents (mikes-agents, new 2026-10-07)

Scheduled ops agents on **.98** that use the local LLM (qwen3.8-27b on the RTX 5090) **only through LiteLLM**.
No cloud models. Code, install docs and diagrams: `~/mikes-agents` on .98 = `github.com/mcropsey/mikes-agents`.

```
 .98 hv-rocky-linux-1 (podman + systemd timers)         GATEWAY (.101)                   MODEL (.194)
 05:30 labinventory ──ssh──► 11 hosts (no LLM)                                       ┌───────────────────┐
 05:40 servicemap  ──ssh──► Grafana's hosts, labels ─┐                                 │                   │
 06:00 secnews     ─┐                                  ┌──────────────┐              │ LM Studio :1234   │
 06:15 labwatch    ─┼── one virtual key + agent ──────►│ LiteLLM :4000├─────────────►│ qwen3.8-27b       │
 06:30 cvewatch    ─┤   registry entry per agent       │ qwen3.8-27b  │              │ RTX 5090          │
 Mon 06:45 patchwatch┘                                  └──────────────┘              └───────────────────┘
       └──► HTML reports on .98 + email to mcropsey@gmail.com (Gmail SMTP, app password)
```

| Agent | Schedule | What it does | Manual run (on .98) |
|---|---|---|---|
| secnews-agent | daily 06:00 | Cybersecurity news briefing: 9 security RSS feeds + new CISA KEV entries, ranked and summarised | `secnews` |
| labwatch-agent | daily 06:15 | Lab health briefing (GREEN/YELLOW/RED). Snapshot of Prometheus/Loki/Alertmanager, then **its own PromQL/LogQL tool calls** to investigate | `labwatch` |
| cvewatch-agent | daily 06:30 | Matches CVEs newly published on NVD + KEV against the software inventory; **emails only if the lab is affected** | `cvewatch` |
| patchwatch-agent | Mon 06:45 | Patch plan: pending updates, Rocky security advisories + CVEs, reboot-needed hosts, broken repos, container images with newer builds. **Reports only; never installs.** | `patchwatch` |
| labinventory | daily 05:30 | (no LLM) SSH inventory of all 11 managed hosts: packages, updates, advisories, containers, k8s images → `/opt/labagents/data/inventory/latest.json` | `labinventory` |

- Files on .98: `/opt/secnews/` and `/opt/labagents/` (code, root-only env files with keys + Gmail app password,
  `data/reports/<agent>/latest.html`), wrappers in `/usr/local/bin/`, units + timers in `/etc/systemd/system/`.
- **LiteLLM:** each agent has its own virtual key (`secnews-agent`, `labwatch-agent`, `cvewatch-agent`,
  `patchwatch-agent`) restricted to `qwen3.8-27b`, 1 request in flight, 30 rpm, linked by `agent_id` to its
  entry on the LiteLLM **Agents** page (4 registered agents).
- **Guardrail:** LiteLLM's default-on `basic-safety-filter` (content filter, created 2026-10-03) blocks normal
  security vocabulary ("attack"). The 4 agent keys carry `metadata.opted_out_global_guardrails:
  ["basic-safety-filter"]`; the filter still applies to every other key.
- GPU use: about 5 min per morning in total (secnews ~1.5 min, labwatch ~1.5 min, cvewatch ~1.5 min), all
  before 07:00; patchwatch adds ~3.5 min on Mondays.

### Agent Lab (retired)

The Agent Lab traffic generator (7 personas on .100/.85/.75/.76, team `agent-lab`, routes `agentlab-main` /
`agentlab-main-patient`, 2026-09-29) is **no longer running**: no `agentlab` containers or services on any host,
no agentlab routes, keys or registry entries in LiteLLM (config backup `config.yaml.bak-pre-agentlab-removal-20261005`
on .101). The `~/agent-lab` directories are still on .101, .100, .85, rk1 and rk2; code + docs remain in
`github.com/mcropsey/agent-lab`.

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
  MCP Host (.102), Logs, **Raspberry Pi Cluster**, **Agent Lab** (incl. *Daily Reports*, hand-written, from mikes-agents),
  **Lab Service Map** (from mikes-agents/servicemap-agent). Daily Reports, Lab Service Map and today's Fleet Overview fix
  are not produced by `gen_dashboards.py`.
- ⚠ `gen_dashboards.py` (rasp5) does **not** generate the *Raspberry Pi Cluster* or *Agent Lab* folders, and its
  documented deploy step runs `rm -rf /opt/monitoring/grafana/dashboards/*` — running it as written deletes both.
  The Agent Lab dashboard has its own generator/installer: `~/agent-lab/grafana/install.sh` (touches only its folder).
- Prometheus config is a mounted **directory** (`prometheus/config/`); reload with `curl -XPOST localhost:9090/-/reload`.
- Alertmanager receiver is still `null` — alerts fire but are **not delivered anywhere**.
- The Grafana admin password in `/opt/monitoring/.env` no longer works (changed in the UI since install).
- Firewalls: .98/.101/.102 use firewalld rich rules (**never** `firewall-cmd --reload` there);
  .99/.100 use nft table `inet monitoring_acl` via `monitoring-acl.service`.
- Loki :3100 on .98 accepts pushes only from lab hosts (.75–.78, .85, .99–.105; firewalld rich rules).
- 2026-10-13: port 9440/tcp (servicemap-web) added on .98 with `firewall-cmd --permanent --add-port` **and a
  `--reload`** (before this note was read). Checked afterwards: Docker published ports (vnotes :8000), container
  egress, Grafana/Prometheus/Loki all fine. Next time use `--add-port` without `--permanent`+reload (runtime) plus
  `--permanent` separately.

---

## Host Summary

| IP | Hostname | Role | OS | RAM (total / avail) | Disk used | Uptime |
|----|----------|------|----|-----|-----------|--------|
| .85 | rasp5 | Control node, Twingate | Ubuntu 24.04.5 (RPi 5) | 7.7 / 5.2 Gi | 16 / 117 G | 16h |
| .98 | hv-rocky-linux-1 | **Observability hub** + **lab agents (mikes-agents)** + VulnNotes API + MCP session host | Rocky 9.8 | 7.5 / 4.9 Gi | 19 / 70 G | 4d |
| .99 | hv-rocky-linux-2 | k3s cluster + Cloudflare ingress | Rocky 9.8 | 3.6 / 1.5 Gi | 15 / 70 G | 4d |
| .100 | hv-rocky-linux-3 | Kong + Jenkins CI | Rocky 9.8 | 7.5 / 4.9 Gi | 15 / 70 G | 12h |
| .101 | hv-rocky-linux-4 | **LiteLLM AI gateway** + crAPI lab + Noname sensor | Rocky 9.8 | 3.6 / 1.2 Gi | 19 / 70 G | 4d |
| .102 | hv-rocky-linux-5 | Noname sensor + MCP servers + AI sim | Rocky 9.8 | 3.6 / 2.1 Gi | 19 / 70 G | 4d |
| .103 | hv-rocky-linux-6 | RHCSA practice workstation | Rocky 10.2 | 3.6 / 2.6 Gi | 6.5 / 70 G | 4d |
| .104 | hv-rocky-linux-7 | RHCSA practice workstation | Rocky 10.2 | 3.6 / 2.5 Gi | 11 / 70 G | 4d |
| .105 | utility.lab.example.com | RHCSA utility server: dnf repos + NFS | Rocky 10.2 | 3.6 / 2.6 Gi | 17 / 70 G | 4d |
| .75 | rk1 | microk8s node | Ubuntu 24.04.4 (RPi 5) | 7.8 / 6.5 Gi | 8.6 / 58 G | 4d |
| .76 | rk2 | microk8s node | Ubuntu 24.04.4 (RPi 5) | 7.8 / 6.4 Gi | 8.7 / 58 G | 4d |
| .77 | rk3 | spare (reserved) | Ubuntu 24.04.4 (RPi 5) | 7.8 / 7.3 Gi | 3.9 / 58 G | 2d |
| .78 | rk4 | spare (reserved) | Ubuntu 24.04.4 (RPi 5) | 7.8 / 7.3 Gi | 4.0 / 58 G | 4d |

Rocky VMs: Hyper-V on an Intel i9-9900K @ 3.60 GHz. Raspberry Pis: Pi 5 Model B, aarch64.
⚠ RAM on .99 (was 6.0 Gi) and .101 (was 8.1 Gi) now reports 3.6 Gi — cause not verified (Hyper-V dynamic
memory or a VM resize). .101 is tight with LiteLLM added; keep new workloads off it.
RAM/disk/uptime columns are from 2026-09-29; every Rocky host reboots on 2026-10-07 22:05+ after patching.

---

## .85 — rasp5

**Role:** Control / management node — Claude Code and opencode run here; holds the monitoring runbook
(`~/MONITORING_SETUP.md`) and staging (`~/monitoring/`).

| Service / container | Description | Status |
|---------|-------------|--------|
| `twingate-jade-limpet` (Docker) | Twingate connector (remote access) | Up (healthy) |
| `node_exporter.service` | :9100, scraped by Prometheus on .98 | Running |
| `alloy.service` | journal + docker logs → Loki | Running |
| `glances.service` / `atop` / `cups` | local monitoring, accounting, printing | Running |

- Docker Compose CLI plugin v5.5.1 installed per-user (`~/.docker/cli-plugins`) for the Agent Lab (2026-09-29; Agent Lab since retired).
- SSH mesh hub: passwordless to .98–.102 and rk1–rk4 with `~/.ssh/id_ed25519`; **.103–.105 still reject this key**
  (they accept the key on .101).

---

## .98 — hv-rocky-linux-1

**Role:** Observability hub (Grafana/Prometheus/Loki/Alertmanager) + **lab agents (mikes-agents)** + VulnNotes API lab target + MCP session host

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

**Lab agents** (rootful **podman**, started by systemd timers; see *AI Gateway & Lab Agents* above):

| Timer → service | Image / command | Data |
|---|---|---|
| `labinventory.timer` 05:30 | `/opt/labagents/app/labinventory.py` on the host as mcropsey (SSH + `sudo -n` to all managed hosts) | `/opt/labagents/data/inventory/` |
| `secnews.timer` 06:00 | `localhost/secnews` | `/opt/secnews/data/reports/` |
| `labwatch.timer` 06:15 · `cvewatch.timer` 06:30 · `patchwatch.timer` Mon 06:45 | `localhost/labagents` (`python -m labagents.<agent>`, `--network host`) | `/opt/labagents/data/reports/<agent>/` |

Secrets: `/opt/secnews/secnews.env`, `/opt/labagents/labagents.env` (root, 600: LiteLLM keys, Gmail app password).
Git clones here: `~/mikes-agents`, `~/cropsey-lab`, `~/vnotes` (push over HTTPS, token in `~/.git-credentials`).
.98 has passwordless SSH + `sudo -n` to .99–.105, rasp5, rk1, rk2 (used by labinventory and the reboot run).

| URL | What |
|-----|------|
| http://192.168.1.98:3000 | Grafana |
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
| `ingress-nginx` controller | — | :80/:443/:8443, status :8181 |
| kube-system: coredns, metrics-server, local-path-provisioner | ClusterIP | — |

- `cloudflared.service` routes tunnel traffic into ingress-nginx. k3s API :6443; kubelet :10250 scraped by Prometheus.
- ⚠ `sudo` drops `/usr/local/bin` — use `sudo /usr/local/bin/k3s kubectl ...`. Firewall: nft `inet monitoring_acl`.

---

## .100 — hv-rocky-linux-3

**Role:** Kong API Gateway (Podman) + Jenkins CI/CD (Docker)

| Container | Runtime | Image | Ports |
|------|---------|-------|-------|
| `kong` | Podman (rootful) | `kong:3.6` | 0.0.0.0:80→8000, :8001 admin |
| `jenkins` | Docker | `myjenkins-noname:2.568.3-jdk21` | :8080, :50000 |
| `jenkins-docker` | Docker | `docker:dind` | :2376 (TLS) |
| `cadvisor` | Docker | `ghcr.io/google/cadvisor:v0.60.6` | :8081 |

Systemd: `docker`, `containerd`, `node_exporter`, `prometheus-podman-exporter` (:9882), `alloy`.
URLs: Jenkins http://192.168.1.100:8080 (`/blue`), Kong proxy :80, Kong admin :8001, cAdvisor :8081.
Rebooted ~2026-09-29 09:00 (all containers came back on their own). Firewall: nft `inet monitoring_acl`.
`jenkins` restart policy changed `on-failure` → `unless-stopped` on 2026-10-07 so it comes back after reboots.
Jenkins image plugins (as of 2026-09-19): blueocean, docker-workflow, github, github-branch-source,
pipeline-model-definition, credentials-binding, plain-credentials, git, workflow-aggregator; JDK21 + Docker CLI.

---

## .101 — hv-rocky-linux-4

**Role:** **LiteLLM AI gateway** + crAPI vulnerable-app lab + Noname Sensor. (Retired Agent Lab repo still in `~/agent-lab`.)

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

The `agentlab-main` / `agentlab-main-patient` routes, team `agent-lab` and its keys were removed with the
Agent Lab (~2026-10-05; backup `config.yaml.bak-pre-agentlab-removal-20261005`).

As of 2026-10-07 LiteLLM runs as **quadlets** (`/etc/containers/systemd/litellm.container` → `litellm.service`,
image pinned to `ghcr.io/berriai/litellm:v1.104.0`; `litellm-db.container` → own Postgres `postgres:16.15`,
network `litellm.network`).

**Agents page:** 4 registered agents (`secnews-agent`, `labwatch-agent`, `cvewatch-agent`, `patchwatch-agent`,
the lab agents on .98), each linked to its own virtual key via `agent_id`; keys allow only `qwen3.8-27b`,
1 request in flight, 30 rpm. Registration script: `~/mikes-agents/litellm/register_agent.py` (runs inside the
container, so the master key never leaves it).

**Guardrail:** `basic-safety-filter` (`litellm_content_filter`, pre_call, **default on**, created 2026-10-03):
blocks the harm/violence/self-harm/child-safety/prompt-injection/exfiltration categories at medium severity.
The 4 lab-agent keys opt out of it via key metadata `opted_out_global_guardrails` (admin-set only); everything
else is still filtered.

### Podman containers (rootful)

| Name | Image | Ports |
|------|-------|-------|
| `litellm` | `ghcr.io/berriai/litellm:v1.104.0` (quadlet `litellm.service`) | :4000 |
| `litellm-db` | `postgres:16.15` (quadlet `litellm-db.service`) | 5432 internal (LiteLLM) |
| `postgresdb` | `postgres:14` | 5432 internal (crAPI) |
| `mongodb` | `mongo:4.4` | 27017 internal |
| `chromadb` | `chromadb/chroma:latest` | 8000 internal |
| `mailhog` | `crapi/mailhog:latest` | :8025 |
| `api.mypremiumdealership.com` | `crapi/gateway-service:latest` | 443 internal |
| `crapi-identity` / `crapi-community` / `crapi-workshop` | `crapi/*` | internal |
| `crapi-chatbot` | `crapi/crapi-chatbot:latest` | :5500 |
| `crapi-web` | `crapi/crapi-web:latest` | :8888, :8443, :30080, :30443 |
| `searxng` | `searxng/searxng:latest` | :8080 |
| `juice-shop` | `bkimminich/juice-shop:latest` | :3000 |
| `dvga` | `dolevf/dvga:latest` (Damn Vulnerable GraphQL App) | :5013 |

Systemd: `noname-sensor` (eBPF), `crapi.service` (starts the crAPI compose stack at boot), `node_exporter`,
`prometheus-podman-exporter`, `alloy`.
URLs: crAPI http://192.168.1.101:8888 / https://…:8443, MailHog :8025, chatbot :5500, SearXNG :8080, Juice Shop :3000,
**LiteLLM :4000**. Firewall: firewalld rich rules — **never** `firewall-cmd --reload`.
Also installed 2026-09-29: `chromium-headless` (EPEL) for rendering these docs to PNG.

⚠ **Noname sensor repo** (`/etc/yum.repos.d/noname-registry.repo`, `[Noname-Artifactory]` on Google Artifact
Registry, credential embedded in the URL) fails since ~2026-10 (repomd.xml can't be downloaded, likely an expired
or rotated credential). Set `skip_if_unavailable=1` on 2026-10-07 so plain `dnf` keeps working (backup
`/root/noname-registry.repo.bak-20261007`). Sensor updates (`noname-sensor-ebpf-rhel9`, installed 3.3-62) need a
fresh repo file from the Noname/Akamai portal. Same on .102.

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
Noname sensor repo: same broken `[Noname-Artifactory]` repo as .101, same `skip_if_unavailable=1` fix (2026-10-07).

---

## .75 / .76 — rk1, rk2 (Raspberry Pi 5)

**Role:** 2-node **microk8s v1.29.15** cluster

| microk8s workload | Status |
|---|---|
| `vampi/vampi` (2 pods) | Running |
| `ingress/nginx-ingress-microk8s-controller` (per node) | Running |
| kube-system: calico, coredns, hostpath-provisioner, metrics-server | Running |

Add-ons: dns, ha-cluster, helm/helm3, hostpath-storage, ingress, metrics-server, storage.
The Agent Lab `agentlab.service` is gone (retired); `~/agent-lab` + `.venv` remain on both nodes.
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
| 2026-10-13 | .98 | **Grafana fix:** *Fleet Overview* per-host rows used `host="$host"`; Grafana regex-escapes multi-value variables, so `utility.lab.example.com` (the only host name with dots) matched nothing and its row was blank. Now `host=~"$host"` in the live dashboard and in `gen_dashboards.py` (.98 `/opt/monitoring/grafana/src/` and rasp5 `~/monitoring/`). The live JSON has hand edits the generator lacks, so it was edited in place, not regenerated. |
| 2026-10-13 | .98 | **Grafana:** new folder *Lab Service Map* (`/d/lab-service-map`); *Agent Lab · Daily Reports* gains servicemap panels, a *Full reports* panel and links from every report log to its HTML report. Prometheus job `servicemap` (`localhost:9440`, 5m, `honor_labels`). **agenthub** (opencode, report exporter :9430) now also covers servicemap. `servicemap-web` serves `/reports/` for all agents (read-only bind mounts; `/opt/secnews` stays root-only). |
| 2026-10-13 | rasp5 | cropseyit.com page: footer link *Lab Agent Reports (LAN only)* → `http://192.168.1.98:9440/reports/` (repo `cropseyit-site`, deployed to `/var/www/html`). |
| 2026-10-13 | .98 | **servicemap-agent** added (mikes-agents): daily 05:40 SSH inventory of every host Grafana monitors (systemd/user units, timers, cron, podman rootless+root, docker, k3s/microk8s, ad-hoc scripts, sockets), labelled by qwen3.8 via its own LiteLLM key; page served by `servicemap-web.service` on **:9440** (firewalld port opened), published to [`service-map/`](service-map/) here. rk3/rk4 (reserved) skipped. |
| 2026-10-13 | .101 | LiteLLM: key + Agents-page entry `servicemap-agent` (qwen3.8-27b only, guardrail opt-out like the other agents). |
| 2026-10-07 | .98–.105 | **Security updates** (`dnf update --security`, unreachable repos skipped): ~365 package installs/upgrades on 8 hosts (logs on .98 count 26–91 per host): new kernels, ghostscript, sudo, openssh, freerdp (Critical, .104), expat, libxml2, curl, nodejs, vim, podman, rsyslog. No security updates left. **Reboots scheduled 22:05** one host at a time (.103 → .104 → .105 → .102 → .100 → .99 → .101 → .98) by `~/patching/lab-reboots.sh` on .98; it stops if a host or its containers don't come back. Log: `~/patching/2026-10-07/reboots.log`. |
| 2026-10-07 | rasp5, rk1, rk2 | **Security updates** (`-security` pocket): `sudo` on all three, `poppler` on rasp5. No reboot needed. |
| 2026-10-07 | .101 .102 | `Noname-Artifactory` dnf repo set `skip_if_unavailable=1` (repo credential appears expired). |
| 2026-10-07 | .100 | `jenkins` container restart policy → `unless-stopped`. |
| 2026-10-07 | .98 | **Lab agents deployed** (mikes-agents): secnews (06:00), labwatch (06:15), cvewatch (06:30), patchwatch (Mon 06:45), labinventory (05:30). Podman images `localhost/secnews`, `localhost/labagents`; `/opt/secnews`, `/opt/labagents`. Repo `github.com/mcropsey/mikes-agents`. |
| 2026-10-07 | .101 | LiteLLM: 4 agent keys + Agents-page entries for the lab agents (qwen3.8-27b only); keys opted out of `basic-safety-filter`. |
| ~2026-10-05 | .100 .85 .75 .76 .101 | **Agent Lab retired**: containers/services gone, LiteLLM agentlab routes, team, keys and agents removed. LiteLLM moved to quadlets with its own Postgres (`litellm-db`) and a pinned image (v1.104.0, ~10-06). |
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
