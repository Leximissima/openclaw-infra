# OpenClaw Deployment — Architecture

> Visual reference for a personal OpenClaw deployment on Hetzner Cloud + Tailscale.
> Describes a single-agent, Slack-primary configuration with WhatsApp self-chat capture,
> a read-only second-brain mirror, and three production workflows.

| # | Diagram | Answers |
|---|---|---|
| 1 | [System architecture](#1-system-architecture) | What talks to what? |
| 2 | [Defense in depth](#2-defense-in-depth) | Why is this safe to put on the public internet? |
| 3 | [Provisioning flow](#3-provisioning-flow) | What happens when `pulumi up` runs? |
| 4 | [Working workflows](#4-working-workflows) | What does the agent actually do day-to-day? |

Source of truth: `pulumi/index.ts`, `ansible/playbook.yml`, and the runtime config on the VPS (`~/.openclaw/openclaw.json`). When the diagrams drift from those, the code wins — update both in the same PR.

**Deployment snapshot:** OpenClaw `2026.5.22` · agent backend `@anthropic-ai/claude-code@2.1.97` (Claude Pro) · single agent (`main`) · Hetzner CX43 in Nuremberg · `@openclaw/slack` + `@openclaw/whatsapp` external plugins from clawhub.

---

## 1. System architecture

**What talks to what.** Every client path enters through Tailscale; nothing is reachable on the public internet. Inside the VPS, the gateway runs as a user-level systemd service on `localhost:18789` and is exposed to the tailnet via Tailscale Serve. Sessions run as Docker containers on a bridge network with two volume mounts: the agent's workspace (`/workspace`, read-write) and a pull-only mirror of an external knowledge vault (`/lexis`, read-only). MCP server containers sit on a separate `codex-proxy-net` so sandbox containers can't reach the credential proxy.

The primary control channel is **Slack** (Socket Mode via the external `@openclaw/slack` plugin). **WhatsApp** is present but locked to self-chat only: the operator's own number is the sole entry on the DM allowlist; all groups are disabled. Bulk community content enters as static `.txt` exports dropped into the workspace via Obsidian Sync, not over the WhatsApp channel.

```mermaid
flowchart LR
    subgraph CLIENTS["Operator devices"]
        direction TB
        LAPTOP["Laptop<br/><i>openclaw CLI<br/>+ browser<br/>+ Slack desktop</i>"]
        PHONE["Phone<br/><i>Slack mobile<br/>+ WhatsApp<br/>(self-chat only)<br/>+ Obsidian</i>"]
    end

    subgraph TAILNET["Tailscale (WireGuard mesh, identity-based)"]
        direction TB
        TS_SERVE["Tailscale Serve<br/><i>wss://&lt;host&gt;.&lt;tailnet&gt;.ts.net</i>"]
    end

    subgraph EXT["Third-party APIs"]
        direction TB
        SLACK_API["Slack API<br/>(Socket Mode WS)"]
        WA_API["WhatsApp Web<br/>(Baileys)"]
        OBS_API["Obsidian Sync<br/>(E2E, EU)"]
        ANTHROPIC["Anthropic API<br/>(Claude Pro)"]
        GH["GitHub<br/><i>workspace push</i>"]
    end

    subgraph VPS["Hetzner VPS · CX43 · Nuremberg"]
        direction TB
        HFW{{"Hetzner cloud firewall<br/><b>deny all inbound</b>"}}
        UFW{{"UFW<br/><i>tailscale0 only</i>"}}

        subgraph HOST["Host (Ubuntu 24.04, user: ubuntu)"]
            direction TB
            GW["<b>openclaw-gateway</b><br/><i>systemd --user</i><br/>localhost:18789"]

            subgraph PLUGINS["External plugins (clawhub)"]
                direction LR
                PL_SLACK["@openclaw/slack"]
                PL_WA["@openclaw/whatsapp<br/><i>allowlist: 1 number<br/>groups: disabled</i>"]
            end

            subgraph DOCKER["Docker"]
                direction TB
                subgraph SBNET["bridge network"]
                    SB1["sandbox<br/>(session 1)"]
                    SB2["sandbox<br/>(session N)"]
                end
                subgraph MCPNET["codex-proxy-net"]
                    MCP_CC["claude-code-mcp"]
                    MCP_QMD["qmd-mcp"]
                    MCP_GH["github-mcp"]
                end
            end

            FS[("~/.openclaw/<br/>workspace · sessions ·<br/>devices · qmd index ·<br/>lexis-readonly")]
            OB_DAEMONS["obsidian-headless<br/><i>main + lexis<br/>(read-only)</i>"]
        end
    end

    LAPTOP -- "https / wss" --> TS_SERVE
    PHONE -. "via third-party<br/>cloud channels" .-> SLACK_API
    PHONE -.-> WA_API
    PHONE -.-> OBS_API

    TS_SERVE --> HFW --> UFW --> GW

    GW <--> PL_SLACK
    GW <--> PL_WA
    GW --> SB1
    GW --> SB2
    GW --> MCP_CC
    GW --> MCP_QMD
    GW --> MCP_GH
    GW <--> FS
    OB_DAEMONS <--> FS

    SB1 -. mount /workspace rw .-> FS
    SB1 -. mount /lexis ro .-> FS
    SB2 -. mount /workspace rw .-> FS

    PL_SLACK <-->|"Socket Mode<br/>(events_api)"| SLACK_API
    PL_WA <-->|"WS"| WA_API
    OB_DAEMONS <-->|"WS"| OBS_API
    MCP_CC -- "LLM API" --> ANTHROPIC
    SB1 -. "git push<br/>(deploy key)" .-> GH

    classDef client fill:#e3f2fd,stroke:#1976d2,color:#0d47a1
    classDef tailnet fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c
    classDef vps fill:#fff3e0,stroke:#f57c00,color:#e65100
    classDef plugin fill:#e8eaf6,stroke:#3949ab,color:#1a237e
    classDef ext fill:#eceff1,stroke:#546e7a,color:#263238
    classDef fw fill:#ffebee,stroke:#c62828,color:#b71c1c
    class LAPTOP,PHONE client
    class TS_SERVE tailnet
    class GW,SB1,SB2,MCP_CC,MCP_QMD,MCP_GH,FS,OB_DAEMONS vps
    class PL_SLACK,PL_WA plugin
    class SLACK_API,WA_API,OBS_API,ANTHROPIC,GH ext
    class HFW,UFW fw
```

**Key invariants:**

- No inbound public ports. The Hetzner firewall has no allow rules.
- The gateway binds `127.0.0.1:18789`; Tailscale Serve is the only proxy.
- Sandbox containers can reach the open internet (for `git push`, web fetch) but **cannot** reach `codex-proxy-net`.
- The `/lexis` mount is bind-mounted **read-only**; the sandbox cannot modify the second-brain vault.
- WhatsApp is the highest-blast-radius channel and is locked at the plugin layer: only the operator's own phone number can deliver a message; all groups are disabled.
- Bulk content from third-party chat groups enters the system as **static file exports**, not via live channel integration.

---

## 2. Defense in depth

**Why putting this on the public internet is fine.** Six independent layers stand between the open internet and any code execution. Each layer fails closed; defeating one does not yield meaningful capability on its own.

```mermaid
flowchart TB
    INTERNET(("Public internet")):::danger

    subgraph L1["① Hetzner cloud firewall — deny all inbound"]
        direction TB
        L1_NOTE["No allow rules.<br/>Port scanners see a black hole."]
        subgraph L2["② Tailscale identity — WireGuard mesh"]
            direction TB
            L2_NOTE["Per-device key + SSO identity.<br/>Stolen IP &ne; access."]
            subgraph L3["③ UFW — only tailscale0 interface"]
                direction TB
                L3_NOTE["Belt-and-braces: even if<br/>something bound to 0.0.0.0,<br/>UFW drops non-Tailscale packets."]
                subgraph L4["④ Gateway device pairing + token"]
                    direction TB
                    L4_NOTE["First connect needs SSH approval.<br/>Token cached per device."]
                    subgraph L5["⑤ Channel allowlists (per plugin)"]
                        direction TB
                        L5_NOTE["WhatsApp: 1 number, groups disabled.<br/>Slack: dmPolicy=pairing.<br/>Telegram: explicit user IDs."]
                        subgraph L6["⑥ Docker sandbox (every session)"]
                            direction TB
                            L6_NOTE["UID 1000, --cap-drop ALL,<br/>no host FS, no gateway config,<br/>writable rootfs ephemeral.<br/>/lexis is read-only."]
                            CORE(("Agent code<br/>execution")):::safe
                        end
                    end
                end
            end
        end
    end

    INTERNET -. blocked .-x L1
    L5 -. "ALL sessions sandboxed<br/>(no escape hatch for web chat)" .-> L6

    classDef danger fill:#ffcdd2,stroke:#b71c1c,color:#b71c1c
    classDef safe fill:#c8e6c9,stroke:#1b5e20,color:#1b5e20
    classDef l1 fill:#ffebee,stroke:#c62828,color:#b71c1c
    classDef l2 fill:#fff3e0,stroke:#ef6c00,color:#e65100
    classDef l3 fill:#fffde7,stroke:#f9a825,color:#f57f17
    classDef l4 fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20
    classDef l5 fill:#ede7f6,stroke:#5e35b1,color:#311b92
    classDef l6 fill:#e3f2fd,stroke:#1565c0,color:#0d47a1
    class L1 l1
    class L2 l2
    class L3 l3
    class L4 l4
    class L5 l5
    class L6 l6
```

**Trust boundaries (outside → in):**

| Layer | What it stops | What still passes |
|---|---|---|
| ① Hetzner FW | All unsolicited inbound TCP/UDP. | Outbound: anywhere. |
| ② Tailscale | Anyone not on the operator's tailnet. | Identified tailnet peers. |
| ③ UFW | Packets from a non-`tailscale0` interface that somehow arrived. | Tailscale traffic. |
| ④ Pairing | Authenticated tailnet peers not on the approve-list. | Approved devices holding the token. |
| ⑤ Channel allowlists | Inbound messages from senders outside the per-channel allowlist (different per channel — WhatsApp is tightest). | Allowlisted senders. |
| ⑥ Sandbox | Sandbox → host FS, sandbox → MCP proxy net, privilege escalation, persistent rootfs writes, writes to `/lexis`. | `/workspace` r/w; `/lexis` r/o; outbound internet via Docker NAT. |

Layer ⑤ is deployment-specific — the channel allowlists encode operator intent (which humans/rooms can address the agent). Other layers are mechanical.

See [SECURITY.md](./SECURITY.md) for the full threat model and writable-rootfs rationale.

---

## 3. Provisioning flow

**What happens when `pulumi up` runs.** Pulumi creates infra, cloud-init does a 3-line bootstrap, then Pulumi triggers `scripts/provision.sh`, which hands off to Ansible. Each Ansible role has a tag; day-2 changes re-run individual roles via `./scripts/provision.sh --tags <tag>`.

Channels that live as external plugins (`@openclaw/slack`, `@openclaw/whatsapp`) are installed via `openclaw plugins install` after the `plugins` role and paired interactively the first time (Slack: pairing code via DM; WhatsApp: QR scan).

```mermaid
flowchart TB
    DEV["./scripts/provision.sh<br/>(or pulumi up)"]:::cmd

    subgraph PULUMI["Pulumi (TypeScript) — pulumi/index.ts"]
        direction TB
        P_FW["createFirewall<br/><i>no inbound rules</i>"]
        P_SRV["createServer<br/><i>CX43 + SSH key + user-data</i>"]
        P_TOK["RandomPassword<br/><i>gateway token</i>"]
        P_KEY["tls.PrivateKey × N<br/><i>workspace deploy keys</i>"]
        P_CMD["command.local.Command<br/><i>triggers ansible on<br/>server.id change</i>"]
    end

    subgraph CLOUDINIT["Cloud-init (user-data.ts) — runs once on first boot"]
        direction LR
        CI1["create ubuntu user<br/>+ sudoers"] --> CI2["install<br/>Tailscale"] --> CI3["tailscale up<br/>--ssh<br/>--authkey"]
    end

    subgraph PROVISION["scripts/provision.sh — reads Pulumi secrets, sets PROVISION_* env"]
        direction TB
        SH1["pulumi stack output<br/>(or PROVISION_* env)"] --> SH2["render dynamic<br/>inventory<br/>(Tailscale IP)"] --> SH3["ansible-playbook<br/>playbook.yml"]
    end

    subgraph ANSIBLE["Ansible roles — playbook.yml (order matters)"]
        direction TB
        R01["<b>system</b> · apt + unattended-upgrades"]
        R02["<b>docker</b> · install + ubuntu→docker group"]
        R03["<b>ufw</b> · deny incoming, allow tailscale0"]
        R04["<b>openclaw</b> · install binary, onboard, enable systemd --user"]
        R05["<b>config</b> · model, sandbox mode, allowlists, auth"]
        R06["<b>agents</b> · non-default agents + channel bindings"]
        R07["<b>telegram</b> · channel config + cron prompts"]
        R08["<b>whatsapp</b> · Baileys channel (if agent uses it)"]
        R09["<b>discord</b> · bot token + guild allowlist"]
        R10["<b>obsidian-headless</b> · ob daemon per workspace"]
        R11["<b>qmd</b> · install + per-agent watchers + GGUF models"]
        R12["<b>plugins</b> · MCP adapter, Codex/Claude/Pi/qmd/GH, deny rules"]
        R13["<b>sandbox</b> · build openclaw-sandbox-custom:latest"]
        R14["<b>workspace</b> · git sync timer + deploy keys"]
        R01 --> R02 --> R03 --> R04 --> R05 --> R06 --> R07 --> R08 --> R09 --> R10 --> R11 --> R12 --> R13 --> R14
    end

    subgraph MANUAL["Out-of-band steps (manual, once per channel)"]
        direction TB
        M1["openclaw plugins install @openclaw/slack<br/>→ DM bot → approve pairing code"]
        M2["openclaw plugins install @openclaw/whatsapp<br/>→ scan QR → lock allowlist"]
        M3["openclaw plugins install lexis read-only<br/>obsidian-headless daemon (pull-only mode)"]
    end

    POST["post_tasks · inject sensitive nested keys (xAI, Discord)<br/>· restart gateway · auto-approve pending device pairings"]:::post

    DEV --> PULUMI
    PULUMI --> CLOUDINIT
    P_CMD -- "on server.id change" --> PROVISION
    PROVISION --> ANSIBLE
    R14 --> POST
    POST --> MANUAL
    MANUAL --> DONE(("Gateway live<br/>+ channels paired")):::safe

    classDef cmd fill:#fff,stroke:#333,stroke-width:2px,color:#000
    classDef pulumi fill:#f3e5f5,stroke:#6a1b9a,color:#4a148c
    classDef ci fill:#e1f5fe,stroke:#0277bd,color:#01579b
    classDef sh fill:#fff8e1,stroke:#f9a825,color:#f57f17
    classDef role fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20
    classDef manual fill:#fff3e0,stroke:#ef6c00,color:#e65100
    classDef post fill:#fce4ec,stroke:#ad1457,color:#880e4f
    classDef safe fill:#c8e6c9,stroke:#1b5e20,color:#1b5e20
    class P_FW,P_SRV,P_TOK,P_KEY,P_CMD pulumi
    class CI1,CI2,CI3 ci
    class SH1,SH2,SH3 sh
    class R01,R02,R03,R04,R05,R06,R07,R08,R09,R10,R11,R12,R13,R14 role
    class M1,M2,M3 manual
```

**Why the order matters (not arbitrary):**

- `docker` before `ufw` — the docker install touches iptables; UFW after means UFW wins where they conflict.
- `openclaw` → `config` → `agents` → `telegram` — agents must exist before any channel can bind chat IDs to them; if `sandbox` ran in between, messages would briefly misroute.
- `qmd` before `plugins` — the plugins role registers `qmd-<agent>` MCP servers and requires the qmd binary to exist.
- `sandbox` near the end — the image build is the slowest step (~minutes); failing earlier roles fail fast.
- `workspace` last — only needed if Pulumi has `workspace*RepoUrl` set; runs git sync timer setup once everything to back up exists.

External plugins are deliberately **not** in Ansible: their pairing step is interactive (Slack pairing code in a DM; WhatsApp QR scan with a 60-second window) and re-running an automation through them risks de-pairing a working channel.

**Day-2 operations** use the same playbook with `--tags`:

```bash
./scripts/provision.sh --tags config           # change model / allowlist / auth
./scripts/provision.sh --tags telegram         # update cron prompts
./scripts/provision.sh --tags sandbox -e force_sandbox_rebuild=true
./scripts/provision.sh --check --diff          # dry run
```

---

## 4. Working workflows

**What the agent actually does day-to-day.** Three production workflows are wired through the gateway. Each has a different inbound trigger, runs the same agent inside the same sandbox, and delivers to a different destination.

```mermaid
flowchart LR
    subgraph IN["Inbound"]
        direction TB
        WA_SELF["WhatsApp self-chat<br/><i>operator's own number</i>"]
        TXT_EXPORT["WhatsApp .txt exports<br/><i>community chats,<br/>dropped via Obsidian Sync</i>"]
        SLACK_DM["Slack DM<br/><i>/ask the agent</i>"]
    end

    subgraph SCHED["Scheduler"]
        direction TB
        CRON["openclaw cron<br/><i>Europe/Berlin</i>"]
    end

    subgraph AGENT["Agent runs (each in a fresh sandbox)"]
        direction TB
        W1["<b>Weekly Debrief</b><br/>reads 4 channel exports<br/>→ 280-word digest<br/>+ 4 themes + links"]
        W2["<b>Inbox Processor</b><br/>reads inbox/raw-*.md<br/>→ classifies entries into<br/>people/follow-ups.md<br/>+ notes/YYYY-MM-DD.md"]
        W3["<b>Inbox Cleanup</b><br/>sweeps processed files<br/>&gt;30 days into<br/>inbox/_archive/YYYY-MM/"]
        W4["<b>Silent Capture</b><br/>writes message to<br/>inbox/raw-YYYY-MM-DD.md<br/>replies 📝"]
        W5["<b>Lookup</b><br/>greps /lexis (ro) +<br/>workspace, answers in DM"]
    end

    subgraph OUT["Delivery"]
        direction TB
        SLACK_DEBRIEF["Slack #weekly-debrief"]
        SLACK_DM_OUT["Slack DM<br/>(operator)"]
        WA_REPLY["WhatsApp reply<br/>(silent: just 📝)"]
        FS_OUT[("~/.openclaw/workspace/<br/>inbox · notes · people")]
    end

    CRON -- "Thu 08:00" --> W1
    CRON -- "Daily 22:00" --> W2
    CRON -- "Sun 23:00" --> W3
    TXT_EXPORT -. "Obsidian Sync<br/>→ workspace fs" .-> FS_OUT
    FS_OUT --> W1
    FS_OUT --> W2

    WA_SELF -- "every inbound message<br/>(allowlist-gated)" --> W4
    W4 --> FS_OUT
    W4 --> WA_REPLY

    SLACK_DM -- "every mention/DM" --> W5

    W1 --> SLACK_DEBRIEF
    W2 --> SLACK_DM_OUT
    W3 --> SLACK_DM_OUT
    W5 --> SLACK_DM_OUT

    classDef in fill:#e3f2fd,stroke:#1976d2,color:#0d47a1
    classDef sched fill:#fce4ec,stroke:#c2185b,color:#880e4f
    classDef agent fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20
    classDef out fill:#fff3e0,stroke:#f57c00,color:#e65100
    class WA_SELF,TXT_EXPORT,SLACK_DM in
    class CRON sched
    class W1,W2,W3,W4,W5 agent
    class SLACK_DEBRIEF,SLACK_DM_OUT,WA_REPLY,FS_OUT out
```

**Workflow summary:**

| Workflow | Trigger | Reads | Writes |
|---|---|---|---|
| Weekly Debrief | Cron · Thu 08:00 | `workspace/WhatsApp History/*.txt` (4 channel exports) | Slack `#weekly-debrief` |
| Inbox Processor | Cron · daily 22:00 | `workspace/inbox/raw-*.md` | `workspace/people/follow-ups.md`, `workspace/notes/YYYY-MM-DD.md`, Slack DM summary |
| Inbox Cleanup | Cron · Sun 23:00 | processed `inbox/raw-*.md` &gt;30 days old | `workspace/inbox/_archive/YYYY-MM/`, Slack DM summary |
| Silent Capture | WhatsApp self-chat message | (nothing — direct file write) | `workspace/inbox/raw-YYYY-MM-DD.md` + 📝 reply |
| Lookup | Slack DM / mention | `/workspace` (r/w) + `/lexis` (r/o) | Slack reply |

**Design notes:**

- **Bulk community content does not flow through the WhatsApp channel.** Static `.txt` exports are dropped via Obsidian Sync into the workspace filesystem and consumed there. The WhatsApp channel itself stays locked to a single sender (the operator's own number). This sidesteps Baileys reliability problems and platform ToS risk.
- **Silent Capture costs ~6 seconds of agent compute per inbound self-chat message** because every inbound triggers a full agent turn (even though the agent only emits 📝). For higher-volume capture this would warrant a custom plugin that writes to file without spinning up the agent.
- **Two Obsidian Sync daemons run on the host**: one bidirectional for the agent's workspace, one **pull-only** for the operator's external knowledge vault (`lexis-readonly`). The pull-only daemon ensures the agent can never write back into the external vault, even if a sandbox is compromised — the mount is read-only at the Docker layer **and** the daemon refuses uploads at the application layer.
- **The Slack adapter converts standard Markdown to Slack mrkdwn** for the operator. Cron prompts emit standard `**bold**` / `*italic*` / `[label](url)`, not Slack mrkdwn syntax.

---

## Updating these diagrams

When the deployment changes:

- New channel → add a node to diagram 1 under "Third-party APIs" with the protocol on the edge; if it introduces a new allowlist surface, update diagram 2's layer ⑤ table.
- New cron job or workflow → add a row to diagram 4's table and a swimlane to the flow.
- New Ansible role → add a node to diagram 3, mention ordering rationale if non-trivial.

Render with any Mermaid preview (VS Code extension, GitHub renders inline). Keep the source readable — diff quality matters more than visual perfection.
