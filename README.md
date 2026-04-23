# Self-Hosted AI Agent with Hermes, Ollama, and Matrix

A complete guide to building a private AI agent that runs on your own hardware, responds via an encrypted chat app, writes notes to Obsidian, and costs nothing to run after setup.

---

## What You Are Building

A stack that looks like this when finished:

- You open **Element** (Matrix chat app) on your PC or phone and message your agent like a chat app
- The agent **thinks using your local GPU** via Ollama — no OpenAI, no Anthropic, no cloud API calls
- The agent writes **structured notes into your Obsidian vault**, synced to all devices in seconds
- Everything runs on **Proxmox** on your home server, isolated in its own VLAN
- The agent **cannot reach your personal devices or home network** — network isolation enforces this at the firewall level

Optional: wire up webhooks so the agent acts automatically when GitHub PRs open, timers fire, or external services send events — without you ever opening a chat.

---

## Architecture Overview

```
YOUR DEVICES  (home network, e.g. 192.168.10.x)
┌────────────────────┐   ┌──────────────────────┐
│ PC                 │   │ Phone                │
│ · Element (chat)   │   │ · Element (chat)     │
│ · Obsidian (notes) │   │ · Obsidian (notes)   │
└─────────┬──────────┘   └──────────┬───────────┘
          │ port 8008 (Matrix)      │ port 22000 (Syncthing)
          ▼                         ▼
  ┌───────────────────────────────────────┐
  │  Router — VLAN Firewall               │
  │  Agent zone → Home zone: BLOCKED      │
  └──────────────────┬────────────────────┘
                     │
  ═══════════════════╪═══════ AGENT ZONE (e.g. 192.168.40.x) ═══
                     │
  PROXMOX ───────────┴──────────────────────────────────────────
  │
  ├── VM: matrix-server  (192.168.40.10)
  │     Private Matrix homeserver (Synapse)
  │
  ├── VM: hermes-agent   (192.168.40.11)
  │     Hermes agent — connects to Matrix, reads/writes vault
  │
  └── LXC: obsidian-sync (192.168.40.12)
        Syncthing + git — vault storage, syncs to all devices

  LLM WORKSTATION  (192.168.40.20)
    GPU · Ollama · port 11434
```

**Key principle:** The agent zone is a security boundary. Your agent can receive messages from you and reach the internet for research. It cannot initiate connections back to your home devices, your PC, or your Proxmox management interface.

### Components

| Component | What it does | Where it runs |
|---|---|---|
| Synapse | Private Matrix homeserver — your messaging server | VM on Proxmox |
| hermes-agent | NousResearch AI agent | VM on Proxmox |
| obsidian-sync | Syncthing + git — vault storage and sync | LXC on Proxmox |
| Ollama | Local LLM inference | GPU workstation (physical machine) |
| Element | Matrix chat client | Your PC and phone |
| Obsidian | Note editor | Your PC and phone |

---

## Prerequisites

- **Proxmox** installed and running on a home server
- **Hermes agent** already installed on a Proxmox VM, connected to Ollama
- **Ollama** running on a machine with a GPU, reachable from the agent VM
- **OpenWrt** (or similar) router with VLAN support
- **Debian 12 ISO** for new VMs (download from [debian.org](https://www.debian.org/distrib/))
- Basic comfort using SSH and a text editor like `nano`

> **IP addresses in this guide** are examples using the `192.168.40.x` subnet for the agent zone and `192.168.10.x` for the home zone. Adapt every IP to match your actual network.

---

## Phase 1 — Matrix Homeserver (Synapse)

Matrix is an open messaging protocol. Synapse is the reference server implementation. You are running your own private server — no account on matrix.org needed, no external federation.

### 1.1 Create the VM in Proxmox

In Proxmox web UI → **Create VM**:

| Setting | Value |
|---|---|
| Name | `matrix-server` |
| OS | Debian 12 (netinst ISO) |
| CPU | 2 cores |
| RAM | 3072 MB |
| Disk | 20 GB |
| Network | Your VLAN 40 bridge, VLAN tag `40` |

Start the VM, open the **Console**, and follow the Debian installer:

- Hostname: `matrix-server`
- Software selection: check only **SSH server** and **standard system utilities** — uncheck everything else

After installation, log in as `root` and set a static IP:

```bash
nano /etc/network/interfaces
```

Find the interface block (usually `ens18`) and change it to:

```
iface ens18 inet static
  address 192.168.40.10
  netmask 255.255.255.0
  gateway 192.168.40.1
```

Apply it:

```bash
systemctl restart networking
ip addr show ens18   # verify the IP appears
```

### 1.2 Install Synapse

```bash
apt update && apt upgrade -y
apt install -y lsb-release wget apt-transport-https

wget -O /usr/share/keyrings/matrix-org-archive-keyring.gpg \
  https://packages.matrix.org/debian/matrix-org-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/matrix-org-archive-keyring.gpg] \
  https://packages.matrix.org/debian/ $(lsb_release -cs) main" \
  > /etc/apt/sources.list.d/matrix-org.list

apt update
apt install -y matrix-synapse-py3
```

When prompted for a **server name**, enter: `matrix.local`

> This becomes part of every user ID on your server (e.g. `@you:matrix.local`). It cannot be changed easily later, so choose carefully.

### 1.3 Configure for Private Use

```bash
nano /etc/matrix-synapse/homeserver.yaml
```

Find and set:

```yaml
# Disable public registration — only you create accounts
enable_registration: false

# Listen on all interfaces inside the agent VLAN
listeners:
  - port: 8008
    tls: false
    type: http
    x_forwarded: false
    bind_addresses: ['0.0.0.0']
    resources:
      - names: [client, federation]
        compress: false
```

Start and enable:

```bash
systemctl enable matrix-synapse
systemctl start matrix-synapse
systemctl status matrix-synapse   # should show "active (running)"
```

### 1.4 Create Accounts

**Your personal account:**

```bash
register_new_matrix_user -c /etc/matrix-synapse/homeserver.yaml \
  http://localhost:8008
# Username: yourusername
# Admin: yes
```

**The agent bot account:**

```bash
register_new_matrix_user -c /etc/matrix-synapse/homeserver.yaml \
  http://localhost:8008
# Username: hermes
# Admin: no
```

Generate a strong random password for the bot:

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

Save this password — you will use it in Phase 3.

Matrix IDs:
- Your account: `@yourusername:matrix.local`
- Agent bot: `@hermes:matrix.local`

### 1.5 Connect Element on Your PC

Install Element from [element.io](https://element.io/get-started) or via Flatpak:

```bash
flatpak install flathub im.riot.Riot
```

Open Element → Sign In → click **Edit** on the homeserver URL → enter `http://192.168.40.10:8008` → log in with your account.

Create a room called `hermes-chat`, invite `@hermes:matrix.local`.

Get the room ID: room Settings → Advanced → **Internal room ID** (looks like `!abc123:matrix.local`). Save this for Phase 3.

**Test:** From your PC terminal:

```bash
curl http://192.168.40.10:8008/_matrix/client/versions
# Should return JSON — if it does, Matrix is working
```

---

## Phase 2 — Obsidian Vault Sync Server

This LXC holds your Obsidian vault. Syncthing propagates changes to all your devices within seconds. Git auto-commits every 15 minutes so you can recover anything.

### 2.1 Create the LXC in Proxmox

First, download a template: Proxmox node → local storage → **CT Templates** → Templates → search `debian-12` → Download.

Then **Create CT**:

| Setting | Value |
|---|---|
| Hostname | `obsidian-sync` |
| Template | debian-12-standard |
| Disk | 50 GB |
| CPU | 1 core |
| RAM | 512 MB |
| Network | VLAN 40 bridge, static IP `192.168.40.12/24`, gateway `192.168.40.1` |

Start the LXC and open its Console.

### 2.2 Install Syncthing and Git

```bash
apt update && apt upgrade -y
apt install -y git curl apt-transport-https

curl -s https://syncthing.net/release-key.txt | \
  gpg --dearmor > /usr/share/keyrings/syncthing-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/syncthing-archive-keyring.gpg] \
  https://apt.syncthing.net/ syncthing stable" \
  > /etc/apt/sources.list.d/syncthing.list

apt update && apt install -y syncthing

# Create a non-root user to run Syncthing
useradd -m -s /bin/bash syncuser

# Create vault directory
mkdir -p /data/vault
chown -R syncuser:syncuser /data/vault

# Enable and start
systemctl enable syncthing@syncuser
systemctl start syncthing@syncuser
```

### 2.3 Allow Remote Access to Syncthing Web UI

By default Syncthing only listens on localhost. Change that so you can manage it from your PC browser:

```bash
systemctl stop syncthing@syncuser
su - syncuser
# Find the config (path varies slightly by version):
find /home/syncuser -name "config.xml"
nano /home/syncuser/.local/state/syncthing/config.xml
```

Find `<address>127.0.0.1:8384</address>` and change it to `<address>0.0.0.0:8384</address>`.

```bash
exit   # back to root
systemctl start syncthing@syncuser
```

Verify from your PC browser: `http://192.168.40.12:8384` — Syncthing web UI should load.

### 2.4 Initialize the Vault

```bash
su - syncuser
cd /data/vault

git init
git config user.email "vault@local"
git config user.name "Vault"

# Create folder structure
mkdir -p me/work me/personal/journal
mkdir -p agent/inbox agent/research agent/scheduled
mkdir -p shared/references shared/decisions

# Placeholder files (git ignores empty folders)
find . -type d -exec touch {}/.gitkeep \;

# Index file
cat > shared/_index.md << 'EOF'
# Vault Index

- **me/** — Your personal notes (only you write here)
- **agent/** — Agent output, awaiting your review
  - **inbox/** — Raw output, review before promoting
  - **research/** — Finished research docs
  - **scheduled/** — Automated task outputs
- **shared/** — Notes you have reviewed and approved
EOF

git add -A
git commit -m "init: vault structure"
exit   # back to root
```

### 2.5 Auto-Commit Every 15 Minutes

```bash
su - syncuser
crontab -e
```

Add at the bottom:

```
*/15 * * * * cd /data/vault && git add -A && git commit -m "auto: $(date +'%Y-%m-%d %H:%M')" 2>/dev/null || true
```

This runs silently forever. Recovery is always one `git checkout` away.

### 2.6 Connect Your PC to Syncthing

On your **PC**:

```bash
sudo apt install syncthing
systemctl --user enable syncthing && systemctl --user start syncthing
```

Open `http://127.0.0.1:8384` in your browser → **Actions → Show ID** — copy the Device ID.

On the **server** Syncthing UI (`http://192.168.40.12:8384`):
1. Add Remote Device → paste your PC's Device ID, name it "PC"
2. Edit the `/data/vault` folder → Sharing tab → check "PC"

On your **PC** Syncthing UI:
1. Accept the incoming connection from the server
2. Accept the shared folder — set local path to e.g. `~/Documents/vault`

Open Obsidian → **Open folder as vault** → select `~/Documents/vault`.

For **Android**: install Syncthing-Fork (F-Droid or Play Store), pair it the same way, point the local path to a folder on your phone, then open Obsidian Android and select that folder as the vault.

### 2.7 Test Sync

1. Create a test note in Obsidian on your PC: `me/personal/sync-test.md`
2. Wait 15–30 seconds
3. Open Obsidian on your phone — the note should appear

Recovery test:

```bash
# SSH into obsidian-sync LXC
su - syncuser
cd /data/vault
git log --oneline -5               # see snapshots
git checkout HEAD -- me/personal/sync-test.md   # restore a deleted file
```

---

## Phase 3 — Connect Hermes to Matrix and the Vault

### 3.1 Find the Hermes Config

SSH into your Hermes VM:

```bash
ssh root@192.168.40.11
```

Locate the config:

```bash
find / -name "config.yaml" 2>/dev/null | grep -i hermes
find / -name ".hermes" -type d 2>/dev/null
ps aux | grep hermes   # see which user runs it
```

Common locations: `~/.hermes/config.yaml` or `/home/hermes/.hermes/config.yaml`. Note the path — you will edit it several times.

### 3.2 Configure the Matrix Connection

Open `config.yaml` and find or add the platforms section:

```yaml
platforms:
  matrix:
    enabled: true
    homeserver: "http://192.168.40.10:8008"
    username: "@hermes:matrix.local"
    password: "YOUR_HERMES_BOT_PASSWORD"
    room_id: "!YOUR_ROOM_ID:matrix.local"
```

Keep secrets out of `config.yaml` if Hermes supports a `.env` file (most versions do):

```bash
nano /home/hermes/.hermes/.env
```

```
MATRIX_PASSWORD=your_hermes_bot_password
MATRIX_HOMESERVER=http://192.168.40.10:8008
MATRIX_USERNAME=@hermes:matrix.local
```

### 3.3 Mount the Vault via NFS

**On the obsidian-sync LXC** (`ssh root@192.168.40.12`):

```bash
apt install -y nfs-kernel-server
nano /etc/exports
```

Add:

```
/data/vault  192.168.40.11(rw,sync,no_subtree_check,no_root_squash)
```

```bash
exportfs -ra
systemctl enable nfs-kernel-server
systemctl start nfs-kernel-server
exportfs -v   # verify the export is listed
```

**On the Hermes VM** (`ssh root@192.168.40.11`):

```bash
apt install -y nfs-common
mkdir -p /mnt/vault

# Test mount
mount -t nfs 192.168.40.12:/data/vault /mnt/vault
ls /mnt/vault   # should show: me/  agent/  shared/

# Make it permanent
echo "192.168.40.12:/data/vault  /mnt/vault  nfs  defaults,_netdev  0  0" >> /etc/fstab
mount -a   # apply without rebooting
```

Test a write:

```bash
echo "# Hello from Hermes" > /mnt/vault/agent/inbox/test.md
```

Within 30 seconds, `test.md` should appear in Obsidian on your PC.

### 3.4 Configure Vault Paths in Hermes

In `config.yaml`:

```yaml
vault:
  base_path: "/mnt/vault"
  inbox_path: "/mnt/vault/agent/inbox"
  research_path: "/mnt/vault/agent/research"
  scheduled_path: "/mnt/vault/agent/scheduled"
  read_path: "/mnt/vault/shared"
```

> The exact key names depend on your Hermes version. If `vault:` doesn't exist, search the config for `memory`, `notes`, or `filesystem` — the concept is the same.

### 3.5 Create a Systemd Service

Make Hermes start automatically and restart itself if it crashes:

```bash
nano /etc/systemd/system/hermes.service
```

```ini
[Unit]
Description=Hermes AI Agent
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=hermes
WorkingDirectory=/home/hermes
ExecStart=/usr/local/bin/hermes
Restart=always
RestartSec=15

[Install]
WantedBy=multi-user.target
```

Replace `ExecStart` with the actual command that starts Hermes on your installation. Check with `hermes --help` or look at how it was set up initially.

```bash
systemctl daemon-reload
systemctl enable hermes
systemctl start hermes
systemctl status hermes
```

### 3.6 End-to-End Test

1. Open Element → hermes-chat room
2. Type: `Hello, are you there?`
3. Hermes should respond within 10–30 seconds
4. Type: `Write a note to my vault about today's date`
5. Within 30 seconds a new file should appear in Obsidian under `agent/inbox/`

If Hermes responds but no note appears, check the NFS mount (`df -h | grep vault` on the Hermes VM) and Syncthing status on the sync LXC.

---

## Phase 4 — Network Isolation (OpenWrt Firewall)

This phase makes the network boundary real. After this, even a misbehaving model cannot reach your personal devices — the router enforces it.

### 4.1 Create a Firewall Zone for the Agent VLAN

In OpenWrt LuCI → **Network → Firewall → Zones → Add**:

| Field | Value |
|---|---|
| Name | `agent` |
| Input | REJECT |
| Output | ACCEPT |
| Forward | REJECT |
| Covered networks | your VLAN 40 interface |

In Inter-Zone Forwarding: allow `agent` → `wan` (internet access). Do not allow `agent` to forward to `lan`, `proxmox`, or any other zone.

### 4.2 Traffic Rules

Go to **Network → Firewall → Traffic Rules** and add each rule:

| Rule name | From | To | Port | Protocol | Action |
|---|---|---|---|---|---|
| `allow-matrix` | lan | agent (192.168.40.10) | 8008 | TCP | ACCEPT |
| `allow-syncthing-sync` | lan | agent (192.168.40.12) | 22000 | TCP+UDP | ACCEPT |
| `allow-syncthing-ui` | lan | agent (192.168.40.12) | 8384 | TCP | ACCEPT |
| `block-agent-to-home` | agent | lan | any | any | REJECT |
| `block-agent-to-proxmox` | agent | proxmox zone | any | any | REJECT |

**Rule order matters.** OpenWrt processes rules top to bottom and stops at the first match. The three ACCEPT rules must appear above the REJECT rules. Drag to reorder if needed.

### 4.3 Verify Every Direction

Run these tests and confirm each result:

```bash
# From the Hermes VM — MUST FAIL (blocked):
ping -c 3 192.168.10.5        # your PC's IP — should timeout

# From the Hermes VM — MUST WORK (allowed):
ping -c 3 192.168.40.10       # Matrix server
ping -c 3 192.168.40.20       # Ollama/GPU machine
curl http://192.168.40.10:8008/_matrix/client/versions   # should return JSON
curl https://api.ipify.org    # should return your public IP

# From your PC — MUST WORK:
curl http://192.168.40.10:8008/_matrix/client/versions
# Browser: http://192.168.40.12:8384 (Syncthing UI should load)
```

If the agent VM can reach your PC (`ping` succeeds when it shouldn't), the `block-agent-to-home` rule is missing or in the wrong position. Fix before proceeding.

### 4.4 Intra-Zone Traffic

Devices within VLAN 40 (agent VM, Matrix VM, sync LXC, GPU workstation) need to communicate freely. In most OpenWrt setups this is automatic within a zone, but verify:

```bash
# From Hermes VM — all of these should succeed:
ping -c 3 192.168.40.10   # Matrix
ping -c 3 192.168.40.12   # vault sync
ping -c 3 192.168.40.20   # Ollama
```

If intra-zone pings fail, edit the `agent` zone in OpenWrt and ensure intra-zone forwarding is set to ACCEPT.

---

## Phase 5 — Local Auxiliary Models with Ollama

Hermes runs up to eight background tasks silently alongside your main conversation. On default settings, these can call cloud APIs — or fall back to your main model — without you realising it.

Since you have Ollama running locally, you can route all of these to local models at zero marginal cost.

### The Eight Background Tasks

| Task | What it does | Fires when |
|---|---|---|
| **Compression** | Summarises old messages when context fills up | Every time context hits the threshold — can fire 10–20×/day on heavy use |
| **Flush memories** | Extracts facts from your conversation into memory files | Every session end (`/new`, `/end`) |
| **Web extract** | Summarises a fetched web page before using it | Every web fetch during research |
| **Vision** | Analyses images and screenshots | When you send images |
| **Session search** | Summarises past sessions when you search history | When searching past chats |
| **Skills hub** | Routes your query to the right installed skill | Every query |
| **MCP dispatch** | Routes tool calls to MCP servers | When using external tools |
| **Approval** | Classifies whether a command is risky | In smart approval mode |

**Cost difference example** (from a real test, 50k token context):

| Model used for compression | Cost per pass | Cost at 15 passes/day × 30 days |
|---|---|---|
| Claude Opus (cloud) | ~$0.13 | ~$58/month |
| Cheap cloud model | ~$0.02 | ~$9/month |
| Local Ollama model | $0.00 | $0/month |

### 5.1 Choose Models

Recommended models for each task (adjust to what you have available):

| Task | Recommended local model | Why |
|---|---|---|
| Compression | `llama3:8b` or `qwen2.5:7b` | Good at summarising, fast enough |
| Flush memories | `llama3:8b` or `phi3:mini` | Small, fast structured extraction |
| Web extract | `qwen2.5:7b` or `mistral:7b` | Good factual preservation |
| Vision | `llava:13b` or `llava:34b` | **Must be a multimodal model** |
| Session search, Skills hub, MCP dispatch, Approval | `phi3:mini` | Tiny classification tasks — speed over power |

Pull what you need:

```bash
# On the machine running Ollama:
ollama pull llama3:8b
ollama pull qwen2.5:7b
ollama pull llava:13b      # vision — multimodal
ollama pull phi3:mini      # fast classification

ollama list                # verify they are available
```

### 5.2 Configure Auxiliary Models in Hermes

Open `config.yaml` on the Hermes VM and find or add the `auxiliary` section. Replace `192.168.40.20` with your actual Ollama machine IP:

```yaml
auxiliary:

  compression:
    base_url: "http://192.168.40.20:11434"
    model: "llama3:8b"
    api_key: "placeholder"
    timeout: 120

  flush_memories:
    base_url: "http://192.168.40.20:11434"
    model: "llama3:8b"
    api_key: "placeholder"
    timeout: 60

  web_extract:
    base_url: "http://192.168.40.20:11434"
    model: "qwen2.5:7b"
    api_key: "placeholder"
    timeout: 90

  vision:
    base_url: "http://192.168.40.20:11434"
    model: "llava:13b"
    api_key: "placeholder"
    timeout: 120

  session_search:
    base_url: "http://192.168.40.20:11434"
    model: "phi3:mini"
    api_key: "placeholder"
    timeout: 30

  skills_hub:
    base_url: "http://192.168.40.20:11434"
    model: "phi3:mini"
    api_key: "placeholder"
    timeout: 30

  mcp_dispatch:
    base_url: "http://192.168.40.20:11434"
    model: "phi3:mini"
    api_key: "placeholder"
    timeout: 30

  approval:
    base_url: "http://192.168.40.20:11434"
    model: "phi3:mini"
    api_key: "placeholder"
    timeout: 30
```

Notes on the fields:
- `base_url` overrides `provider` — Hermes sends requests directly to your Ollama endpoint
- `api_key: "placeholder"` — Ollama does not require a real key, but the field must not be empty
- Increase `timeout` if you see timeout errors — local inference can be slower than cloud APIs
- Model names must exactly match `ollama list` output

Restart Hermes:

```bash
systemctl restart hermes
journalctl -u hermes -f   # watch logs for aux task activity
```

### 5.3 About Memory Files

Hermes stores long-term memory in plain Markdown files you can read and edit directly:

```bash
cat /home/hermes/.hermes/memory.md   # facts hermes has learned
cat /home/hermes/.hermes/user.md     # what hermes knows about you
```

You can add context to `user.md` manually — Hermes uses it on every query. The more accurate this file is, the better the agent performs for your specific use case.

---

## Phase 6 — Webhooks and Automation

With the setup so far, Hermes only acts when you message it. Webhooks change that — external events trigger the agent automatically.

### How Webhooks Work

```
External service (GitHub, Stripe, a cron timer...)
  │
  │ POST https://your-public-url/webhook/route-name
  ▼
[ngrok or similar tunnel] → Hermes gateway (port 8644)
  │
  │ validates signature → renders prompt → runs agent
  ▼
Agent works, sends result to your Matrix room
```

There are two directions:

- **Inbound (world → agent):** External service POSTs to your agent. Requires a public URL (ngrok or equivalent).
- **Outbound (agent → world):** Agent makes HTTP calls to external services. No public URL needed — the agent initiates.

### 6.1 Enable the Webhook Gateway

Generate a secret key:

```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
# Save the output — you'll use it in Hermes config and in each external service
```

Add to `.env`:

```
WEBHOOK_ENABLED=true
WEBHOOK_PORT=8644
WEBHOOK_SECRET=your_generated_secret_here
```

Add to `config.yaml`:

```yaml
platforms:
  webhook:
    enabled: true
    port: 8644
```

Restart Hermes and verify:

```bash
systemctl restart hermes
ss -tlnp | grep 8644   # should show LISTEN on 0.0.0.0:8644
```

### 6.2 Set Up a Public Tunnel (ngrok)

External services need a public HTTPS URL pointing to your home network. ngrok provides this.

Install ngrok from [ngrok.com](https://ngrok.com/download), authenticate, then start a tunnel:

```bash
# Tunnel from public internet → your Hermes VM's webhook port
ngrok http 192.168.40.11:8644
```

ngrok shows a public URL like `https://abc123.ngrok-free.app`. Copy it — you'll register it with external services.

> **Free tier caveat:** The URL changes every time you restart ngrok. Paid plans give you a stable URL, which is worth it once you have more than one or two webhooks set up.

Keep ngrok running in a persistent session:

```bash
tmux new -s ngrok
ngrok http 192.168.40.11:8644
# Detach: Ctrl+B then D
# Reattach later: tmux attach -t ngrok
```

Or run it as a systemd service on your PC.

### 6.3 Example: GitHub Pull Request Auto-Review

**Install GitHub CLI on the Hermes VM:**

```bash
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | \
  dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] \
  https://cli.github.com/packages stable main" | \
  tee /etc/apt/sources.list.d/github-cli.list > /dev/null
apt update && apt install -y gh
gh auth login
```

**Add a route in `config.yaml`:**

```yaml
platforms:
  webhook:
    enabled: true
    port: 8644
    routes:
      - name: github-pr
        path: /webhook/github-pr
        deliver_to: matrix
        prompt: |
          A pull request was opened.
          Repository: {{repository}}
          PR: #{{number}} — {{title}}
          Author: {{author}}
          URL: {{url}}

          Use the GitHub CLI to fetch the diff, review the changes for
          correctness and risks, and post a review comment on the PR.
```

**Register the webhook in GitHub:**

Go to your repository → Settings → Webhooks → Add webhook:

- Payload URL: `https://your-ngrok-url.ngrok-free.app/webhook/github-pr`
- Content type: `application/json`
- Secret: the same secret you generated in Step 6.1
- Events: select **Pull requests** only

After saving, GitHub fires a ping — check `journalctl -u hermes -f` to confirm it was received.

**Test it:** Open a PR on that repository. Within 1–2 minutes, Hermes should post a review comment automatically.

### 6.4 Example: Daily Briefing Written to Obsidian (Cron, Outbound)

No public URL needed — the agent initiates this on a schedule.

Create a news fetch script:

```bash
mkdir -p /home/hermes/scripts
cat > /home/hermes/scripts/fetch_news.py << 'SCRIPT'
#!/usr/bin/env python3
import urllib.request, xml.etree.ElementTree as ET, json, sys

FEEDS = [
    "https://feeds.bbci.co.uk/news/technology/rss.xml",
    "https://hnrss.org/frontpage",
    # Add any RSS feeds relevant to your interests
]

articles = []
for url in FEEDS:
    try:
        req = urllib.request.urlopen(url, timeout=10)
        for item in ET.parse(req).getroot().iter('item'):
            title = item.findtext('title', '').strip()
            desc  = item.findtext('description', '').strip()
            link  = item.findtext('link', '').strip()
            if title:
                articles.append({"title": title, "description": desc[:300], "link": link})
    except Exception as e:
        print(f"Error: {url}: {e}", file=sys.stderr)

print(json.dumps(articles[:40], indent=2))
SCRIPT
chmod +x /home/hermes/scripts/fetch_news.py
```

Add a scheduled task in `config.yaml`:

```yaml
cron:
  tasks:
    - name: morning-briefing
      schedule: "0 8 * * *"      # 8am daily (UTC — adjust for your timezone)
      script: /home/hermes/scripts/fetch_news.py
      prompt: |
        Analyse today's news from the script output.
        Identify the 10 most significant stories, grouped by category.
        Write a concise briefing as a Markdown document.
        Save it to: /mnt/vault/agent/scheduled/{{date}}-briefing.md
```

Restart Hermes. Trigger a manual test run:

```bash
hermes cron run morning-briefing
# or: hermes cron list  →  hermes cron run <number>
```

Check Obsidian — the briefing note should appear in `agent/scheduled/` within a minute.

### 6.5 Example: Agent Posts Analysis to Your Website (Outbound Webhook)

If you have a web app with an API endpoint, the agent can POST structured data to it on a schedule — no user action required.

Store the endpoint and a shared secret in `.env`:

```
MY_APP_WEBHOOK_URL=https://your-site.com/api/agent-updates
MY_APP_WEBHOOK_SECRET=shared_secret_between_hermes_and_your_app
```

In the cron task prompt, instruct the agent explicitly:

```yaml
prompt: |
  Analyse the script output.
  Return only valid JSON matching this shape:
  {"timestamp": "...", "summary": "...", "items": [...]}

  Then use your terminal tool to POST the JSON to {{env.MY_APP_WEBHOOK_URL}}
  with header X-Webhook-Secret: {{env.MY_APP_WEBHOOK_SECRET}}
  Report the HTTP status code in your reply.
```

Hermes uses its built-in terminal tool to make the HTTP call. No ngrok, no inbound tunnel needed.

---

## Troubleshooting Reference

| Symptom | Check | Fix |
|---|---|---|
| Hermes doesn't respond in Element | `systemctl status hermes` | `systemctl restart hermes` — check logs |
| Element can't connect to homeserver | `curl http://192.168.40.10:8008` from PC | Check firewall rule `allow-matrix` |
| Obsidian vault not syncing to phone | Syncthing-Fork app status | Allow background activity in Android battery settings |
| Hermes can't write to vault | `df -h \| grep vault` on Hermes VM | `mount /mnt/vault` or fix NFS |
| Synapse won't start | `ss -tlnp \| grep 8008` | Kill conflicting process, check logs |
| Agent can ping your PC (bad!) | Test from Hermes VM | Add/fix `block-agent-to-home` in OpenWrt |
| Aux tasks still using cloud | Hermes logs show provider name | Restart Hermes after config change, check for `provider: auto` overriding `base_url` |
| Webhook not received | ngrok still running? URL same as registered? | Check `journalctl -u hermes`, use GitHub "Redeliver" button |
| Local model timeout on aux tasks | Model too slow for the window | Increase `timeout` in config, try a smaller model |

---

## Quick Reference

```
SERVICES
  Matrix homeserver:   http://192.168.40.10:8008
  Syncthing web UI:    http://192.168.40.12:8384
  Ollama API:          http://192.168.40.20:11434
  Hermes webhook port: 192.168.40.11:8644

USEFUL COMMANDS (on respective VMs)
  Hermes logs (live):     journalctl -u hermes -f
  Restart Hermes:         systemctl restart hermes
  Restart Matrix:         systemctl restart matrix-synapse  (on matrix-server VM)
  Restart Syncthing:      systemctl restart syncthing@syncuser  (on obsidian-sync LXC)
  Vault git history:      cd /data/vault && git log --oneline -20
  Restore deleted file:   git checkout HEAD -- path/to/file.md
  Check Ollama models:    ollama list  (on GPU machine)
  Manual cron trigger:    hermes cron run <task-name>
  Agent memory files:     cat ~/.hermes/memory.md  /  cat ~/.hermes/user.md
```

---

## What to Build Next

Once the core is running, these extensions are straightforward to add:

**Remote access from mobile data** — Install Tailscale on your Matrix VM, sync LXC, and phone. Update Element's homeserver URL to the Tailscale IP. You can now reach your agent from anywhere without exposing ports to the internet.

**Multiple LLM backends** — Keep Ollama for private tasks. Add an external API (Anthropic, Groq, etc.) for tasks that need more capability. Tools like [LiteLLM](https://github.com/BerriAI/liteLLM) or [openrouter.ai](https://openrouter.ai) act as a unified proxy in front of multiple providers.

**More webhook triggers** — Any service with a webhook URL field can trigger your agent: Stripe payments, Home Assistant events, cal.com meeting bookings, Sentry errors, custom Typeform submissions. The pattern is always the same: add a route in config, register the URL in the service.

**Additional agents** — Each agent gets its own subfolder in the vault (`/data/vault/agent-2/`), its own Matrix account, and its own VM. They coordinate through shared files in `shared/`.

---

## Resources

- [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) — Hermes agent source and config reference
- [Proxmox documentation](https://pve.proxmox.com/pve-docs/)
- [Matrix/Synapse documentation](https://matrix-org.github.io/synapse/latest/)
- [Syncthing documentation](https://docs.syncthing.net/)
- [OpenWrt firewall docs](https://openwrt.org/docs/guide-user/firewall/firewall_configuration)
- [ngrok documentation](https://ngrok.com/docs)
- [Ollama model library](https://ollama.com/library)
