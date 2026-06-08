<div align="center">

# 🤖 Chat with Claude Code directly from Slack

### I built a Slack bot that runs **Claude Code** from n8n, returns text responses, and uploads files automatically

[![n8n](https://img.shields.io/badge/n8n-Workflow-EA4B71?style=for-the-badge&logo=n8n&logoColor=white)](https://n8n.io)
[![Claude Code](https://img.shields.io/badge/Claude_Code-CLI-D97757?style=for-the-badge&logo=anthropic&logoColor=white)](https://www.anthropic.com)
[![Slack](https://img.shields.io/badge/Slack-Bot-4A154B?style=for-the-badge&logo=slack&logoColor=white)](https://slack.com)
[![WSL2](https://img.shields.io/badge/WSL2-Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)](https://ubuntu.com)
[![Docker](https://img.shields.io/badge/Docker-Container-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com)

</div>

---

## 📋 Table of contents

- [✨ What does this project do?](#-what-does-this-project-do)
- [🏗️ Architecture](#️-architecture)
- [🧩 Components](#-components)
- [⚙️ Step-by-step installation](#️-step-by-step-installation)
  - [1. n8n in Docker](#1-n8n-in-docker)
  - [2. Cloudflare Quick Tunnel (SSL)](#2-cloudflare-quick-tunnel-ssl)
  - [3. WSL2 + Ubuntu](#3-wsl2--ubuntu)
  - [4. Claude Code on Ubuntu](#4-claude-code-on-ubuntu)
  - [5. SSH server on WSL2](#5-ssh-server-on-wsl2)
  - [6. NIM Proxy for NVIDIA (optional)](#6-nim-proxy-for-nvidia-optional)
  - [7. Neon DB via MCP with DBHub (optional)](#7-neon-db-via-mcp-with-dbhub-optional)
- [🔄 The n8n workflow](#-the-n8n-workflow)
- [🚀 Starting from scratch](#-starting-from-scratch)
- [🩺 Troubleshooting](#-troubleshooting)

---

## ✨ What does this project do?

A **conversational agent accessible from Slack** that:

- 💬 Receives a text message from a Slack channel
- 🧠 Processes it with **Claude Code** running on Ubuntu (WSL2) on the same Windows machine
- 📎 Automatically detects if Claude generated a file as output
  - **If there's a file** → downloads it and uploads it to the Slack channel
  - **If there's no file** → sends the response as a text message
- 🗄️ Claude Code has access to a **PostgreSQL database on Neon** via MCP
- 💸 Optionally redirects API calls to **NVIDIA NIM** through a [custom proxy](https://github.com/agus-osilio/nvidia-nim-proxy-for-claude-code) to reduce costs

---

## 🏗️ Architecture

```text
        Slack (incoming message)
                  │
                  ▼
        n8n (Docker on Windows)
                  │  SSH
                  ▼
            Ubuntu WSL2
                  │
                  ▼
          Claude Code CLI
          ├── NVIDIA NIM Proxy   (http://localhost:8082)
          └── DBHub MCP ───────► Neon PostgreSQL
                  │
                  ▼
        n8n processes the output
          ├── File found? ──► SSH Download ──► Slack Upload
          └── Text only?  ──► Slack Send Message
```

---

## 🧩 Components

| Component | Role | Where it runs |
|---|---|---|
| **Slack** | Chat interface with the user (trigger + responses) | Cloud |
| **n8n** | Workflow orchestrator | Docker on Windows |
| **Cloudflare Tunnel** | Exposes n8n with valid HTTPS for the Slack webhook | Windows (PowerShell) |
| **WSL2 + Ubuntu** | Linux system where Claude Code lives | Windows |
| **Claude Code CLI** | The agent that processes prompts | Ubuntu |
| **SSH (OpenSSH)** | Bridge between n8n and Claude Code | Ubuntu |
| [**NIM Proxy**](https://github.com/agus-osilio/nvidia-nim-proxy-for-claude-code) *(optional)* | Redirects calls to NVIDIA NIM | Ubuntu (`:8082`) |
| **DBHub MCP** *(optional)* | Gives Claude access to Neon PostgreSQL | Ubuntu |

---

## ⚙️ Step-by-step installation

### 1. n8n in Docker

> ⚠️ **Important:** set the `WEBHOOK_URL` variable, otherwise n8n shows `http://localhost:5678/...` instead of your public domain.

```powershell
# Stop and remove the existing container (if any)
docker stop n8n
docker rm n8n

# Recreate with the correct WEBHOOK_URL (single line for PowerShell)
docker run -it --name n8n -p 5678:5678 -e WEBHOOK_URL=https://your-domain.com/ -v n8n_data:/home/node/.n8n n8nio/n8n
```

> 💡 The data in the `n8n_data` volume is preserved when the container is removed. Only the container is lost, not the data.

Open n8n in the browser:

```
http://localhost:5678
```

---

### 2. Cloudflare Quick Tunnel (SSL)

Slack requires **HTTPS with a valid SSL certificate** for webhooks. A home PC doesn't have SSL, so we use a Cloudflare Quick Tunnel: free, no account needed, no prior setup.

> ❌ **Don't use n8n's `--tunnel` flag:** it was officially discontinued in March 2026 and fails to start on v1+.

**Installation** — download `cloudflared-windows-amd64.exe` from:
👉 https://github.com/cloudflare/cloudflared/releases/latest

**Usage:**

```powershell
.\cloudflared-windows-amd64.exe tunnel --url http://localhost:5678
```

This generates a public URL like:

```
https://something-random.trycloudflare.com
```

> ⚠️ **The URL changes every time you restart the tunnel** and closes when you close the PowerShell window. You'll need to update it in Slack each time.

**Update the webhook in Slack:** go to the Slack app → **Event Subscriptions** → replace the URL:

```
https://something-random.trycloudflare.com/webhook/[webhook-ID]
```

> The `/webhook/[webhook-ID]` part doesn't change — only the Cloudflare domain does.

---

### 3. WSL2 + Ubuntu

```powershell
# Check if WSL2 is already installed
wsl --list --verbose

# Install Ubuntu
wsl --install -d Ubuntu
```

> 🔑 During installation you'll be asked to create a Unix username and password. **Save them** — they're used for SSH from n8n.

<details>
<summary>📦 Install Ubuntu on a different drive (optional)</summary>

```powershell
# 1. Install normally first
wsl --install -d Ubuntu

# 2. Export to a tar file
wsl --export Ubuntu D:\ubuntu-backup.tar

# 3. Unregister from drive C
wsl --unregister Ubuntu

# 4. Import to the desired drive
mkdir D:\WSL
wsl --import Ubuntu D:\WSL\ D:\ubuntu-backup.tar
```

</details>

Access Ubuntu:

```powershell
wsl -d Ubuntu
```

---

### 4. Claude Code on Ubuntu

```bash
# Update base packages
sudo apt update && sudo apt upgrade -y

# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash - && sudo apt-get install -y nodejs

# Update npm
sudo npm install -g npm@<your-preferred-version>

# Install Claude Code
sudo npm install -g @anthropic-ai/claude-code
```

> ⚠️ `sudo` is required to avoid `EACCES: permission denied` when installing globally.

**Authenticate with Anthropic** (follow the browser flow):

```bash
claude
```

**Create the working directory** — this is where Claude generates files and serves as the working directory for n8n's SSH node:

```bash
mkdir ~/claude-workspace
cd ~/claude-workspace
```

<details>
<summary>🧰 Copy skills from Windows (optional)</summary>

```bash
cp -r /mnt/c/Users/<YOUR_USERNAME>/.claude/skills ~/.claude/
```

</details>

---

### 5. SSH server on WSL2

n8n (in Docker) communicates with Claude Code (in WSL2) via SSH.

```bash
# Install OpenSSH Server
sudo apt install openssh-server -y

# Start the server
sudo service ssh start

# Verify it's running
sudo service ssh status

# Get the WSL2 IP (used as Host in n8n)
hostname -I
```

**SSH credentials in n8n:**

| Field | Value |
|---|---|
| **Host** | IP obtained with `hostname -I` |
| **Port** | `22` |
| **Username** | the user created when installing Ubuntu (e.g. `wsl2_ubuntu`) |
| **Password** | the password created during installation |

---

### 6. NIM Proxy for NVIDIA (optional)

Redirects Claude Code calls to **NVIDIA NIM** models instead of the Anthropic API, to reduce costs.

> 📦 You can find the repository here: [**nvidia-nim-proxy-for-claude-code**](https://github.com/agus-osilio/nvidia-nim-proxy-for-claude-code)

The proxy consists of 3 files: `nim_proxy.py`, `requirements.txt`, and `.env`.

```bash
# Copy files from Windows to WSL2
mkdir ~/claude-proxy
cp /mnt/c/Users/<YOUR_USERNAME>/Desktop/claude\ proxy/* ~/claude-proxy/
cp /mnt/c/Users/<YOUR_USERNAME>/Desktop/claude\ proxy/.env ~/claude-proxy/

# Install Python 3.12
sudo apt install python3.12 python3.12-venv python3-pip -y

# Install dependencies
cd ~/claude-proxy
pip install -r requirements.txt --break-system-packages

# Start the proxy (listens on http://localhost:8082)
python3 nim_proxy.py
```

> 💡 The `\` before the space in `claude\ proxy` is needed because the folder name contains a space.

**Environment variables for Claude Code to use the proxy:**

```bash
export ANTHROPIC_BASE_URL="http://localhost:8082"
export ANTHROPIC_AUTH_TOKEN="your-api-key-here"
```

---

### 7. Neon DB via MCP with DBHub (optional)

Gives Claude Code direct access to the schema and data of a PostgreSQL database on Neon to answer queries in natural language.

```bash
# Install dependencies in the working directory
cd ~/claude-workspace
npm install @bytebase/dbhub pg

# Test the connection
node node_modules/@bytebase/dbhub/dist/index.js \
  --dsn "postgres://user:password@host:5432/db_name?sslmode=require"

# Register the database in Claude Code via MCP
claude mcp add --scope project --transport stdio neon_db -- \
  node ~/claude-workspace/node_modules/@bytebase/dbhub/dist/index.js \
  --dsn "postgres://user:password@host:5432/db_name?sslmode=require"
```

> ⚠️ Neon requires `?sslmode=require` at the end of the DSN. Without it the connection fails with an insecure SSL error.

---

## 🔄 The n8n workflow

```text
Slack Trigger
    ↓
CLAUDE CODE (SSH Execute Command)
    ↓
Code in JavaScript (parse output)
    ↓
IF (file found?)
    ├── False → Send a message (text only)
    └── True  → Download a file (SSH) → Upload a file (Slack)
```

### 🟦 Node 1 — Slack Trigger
- **Event:** Any Event (or Message)
- **Channel:** the channel the bot listens to
- **Key output:** `$json.text` — the text of the received message

### 🟧 Node 2 — CLAUDE CODE (SSH Execute Command)

**Working Directory:**
```
/home/wsl2_ubuntu/claude-workspace
```

**Command** (enable **expression mode** with the `=` icon):
```bash
export ANTHROPIC_BASE_URL="http://localhost:8082" && export ANTHROPIC_AUTH_TOKEN="your-api-key" && BEFORE=$(ls ~/claude-workspace | sort) && claude -p '{{ $json.text }}' --dangerously-skip-permissions && AFTER=$(ls ~/claude-workspace | sort) && NEW=$(comm -13 <(echo "$BEFORE") <(echo "$AFTER") | head -1 | tr -d '[:space:]') && if [ -z "$NEW" ]; then echo "FILE:none"; else echo "FILE:$NEW"; fi
```

**What this command does:**
1. Sets environment variables for the NIM proxy
2. Saves the file list **before** running Claude
3. Runs Claude with the prompt received from Slack
4. Saves the file list **after**
5. Compares both lists with `comm` to detect new files
6. Prints `FILE:filename.ext` if a new file was created, or `FILE:none` if not

> 💡 Use single quotes `'` around `{{ $json.text }}` to avoid conflicts with the double quotes in the bash command.

### 🟨 Node 3 — Code in JavaScript

```javascript
const output = $input.item.json.stdout;
const lines = output.split('\n');
const fileLine = lines.find(line => line.trim().startsWith('FILE:'));
const file = fileLine ? fileLine.replace('FILE:', '').trim() : 'none';
const text = lines.filter(line => !line.trim().startsWith('FILE:')).join('\n').trim();
const hasFile = file !== 'none';

return { json: { file, text, hasFile } };
```

**Output:** `file` (name or `"none"`), `text` (clean response), `hasFile` (boolean).

### 🔀 Node 4 — IF
- **Value:** `{{ $json.hasFile }}`
- **Type:** `Boolean` · **Operation:** `is true`
- **False** → Send a message · **True** → Download a file

> 💡 Using the **Boolean** type (instead of comparing strings) avoids issues with `typeValidation: strict`.

### 🟩 Node 5 — Send a message (Slack)
- **Channel:** destination channel · **Message Text:** `{{ $json.text }}`

### 🟩 Node 6a — Download a file (SSH)
- **Resource:** File · **Operation:** Download
- **Path:** `/home/wsl2_ubuntu/claude-workspace/{{ $json.file }}` (expression mode)
- **File Property:** `data`

### 🟩 Node 6b — Upload a file (Slack)
- **Resource:** File · **Operation:** Upload · **File Property:** `data`
- **Options → Channel:** destination channel

---

## 🚀 Starting from scratch

Every time you restart your PC or close the terminals:

```text
1. Docker Desktop    → verify the `n8n` container is Running
2. n8n               → open http://localhost:5678
3. Cloudflare Tunnel → .\cloudflared-windows-amd64.exe tunnel --url http://localhost:5678
                       → copy the new URL and update it in Slack
4. Ubuntu            → wsl -d Ubuntu
5. SSH               → sudo service ssh start
6. NIM Proxy         → (2nd terminal) cd ~/claude-proxy && python3 nim_proxy.py
```

```powershell
# 1 · n8n (if not running)
docker start n8n

# 3 · Cloudflare Tunnel
.\cloudflared-windows-amd64.exe tunnel --url http://localhost:5678

# 4 · Ubuntu
wsl -d Ubuntu
```

```bash
# 5 · SSH
sudo service ssh start && sudo service ssh status

# 6 · NIM Proxy (second Ubuntu terminal, keep it open)
cd ~/claude-proxy && python3 nim_proxy.py
```

✅ **Done.** Send a message to the bot in Slack and verify it responds.

<details>
<summary>🔌 Shutting down WSL2</summary>

```powershell
wsl --shutdown
wsl --terminate Ubuntu   # or your distro's name
wsl --list --running
```

</details>

> ⚠️ **Reminders**
> - The Cloudflare URL **changes every time** the tunnel is restarted → update it in Slack.
> - If you close the tunnel terminal, the bot stops working.
> - If you close the NIM proxy terminal, Claude Code can't use NVIDIA models.
> - If SSH is not active, n8n can't connect to Claude Code and the workflow fails.

---

## 🩺 Troubleshooting

| Error | Cause | Solution |
|---|---|---|
| `"Darn, that URL doesn't have a valid SSL certificate"` in Slack | The webhook doesn't have valid HTTPS | Use Cloudflare Quick Tunnel |
| n8n's `--tunnel` flag doesn't work | Discontinued in March 2026 | Use Cloudflare Quick Tunnel |
| `EACCES: permission denied` when installing Claude Code | npm without global permissions | `sudo npm install -g @anthropic-ai/claude-code` |
| `connection is insecure (try using sslmode=require)` in DBHub | Neon requires SSL | Add `?sslmode=require` to the DSN |
| `bash: unexpected EOF while looking for matching '"'` | Double quote conflict in the SSH command | Use single quotes: `claude -p '...'` |
| The IF node sends everything through the same branch | Comparing strings with `typeValidation: strict` fails with invisible characters | Use the boolean field `hasFile` |
| The SSH Download node says "No such file" | File Property had the full path | File Property must be just `data` |
| Claude receives `{{ $json.text }}` as a literal string | The Command field is not in expression mode | Enable expression mode (`=` icon) |
| The NIM proxy can't find the module | Dependencies not installed | `pip install -r requirements.txt --break-system-packages` |

---

<div align="center">

### 🛠️ Stack

`Slack` · `n8n` · `Docker` · `Cloudflare Tunnel` · `WSL2 / Ubuntu` · `Claude Code` · `OpenSSH` · `NVIDIA NIM` · `Neon PostgreSQL` · `MCP / DBHub`

</div>
