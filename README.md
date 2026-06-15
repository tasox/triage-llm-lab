# AI-Powered Malware Analysis Lab — Installation Guide

A four-VM isolated sandbox that combines Claude Code with REMnux analysis tools, SIFT forensic workstation, GhidraMCP headless decompilation, and VirusTotal intelligence — all orchestrated through MCP servers from a clean Ubuntu host.

| | |
|---|---|
| **Hypervisor** | VMware Workstation / Fusion |
| **VMs** | Ubuntu + REMnux + SIFT + Windows (future) |
| **LLM** | Claude Code |

---

## Table of Contents

1. [Lab Architecture](#01-lab-architecture)
2. [Prerequisites](#02-prerequisites)
3. [VM1 — Ubuntu 24.04 (LLM Host)](#03-vm1--ubuntu-2404-llm-host)
4. [VM2 — REMnux (Analysis Machine)](#04-vm2--remnux-analysis-machine)
5. [VM3 — SIFT Workstation (Forensics)](#05-vm3--sift-workstation-forensics)
6. [Network Configuration](#06-network-configuration)
7. [SSH Key Setup](#07-ssh-key-setup)
8. [remnux-ssh MCP Server](#08-remnux-ssh-mcp-server)
9. [VirusTotal MCP Server](#09-virustotal-mcp-server)
10. [Ghidra MCP (Docker Headless)](#10-ghidra-mcp-docker-headless)
11. [sift-ssh MCP Server](#11-sift-ssh-mcp-server)
12. [REMnux Analysis Tools](#12-remnux-analysis-tools)
13. [SIFT Forensic Tools](#13-sift-forensic-tools)
14. [MCP Tool Groups](#14-mcp-tool-groups)
15. [Analysis Workflows](#15-analysis-workflows)
16. [Troubleshooting](#16-troubleshooting)
17. [Security Notes](#17-security-notes)

---

## 01. Lab Architecture

Four VMs with strict network isolation. Only VM1 has internet access. Samples never touch the LLM host.

```
                          🌐 Internet / Anthropic API
                                    │
   ┌──────────────────┐   SSH / Host-only    ┌──────────────────────┐
   │ VM1               │   192.168.56.0/24    │ VM2                   │
   │ Ubuntu 24.04      │ ───────────────────► │ REMnux                │
   │ Claude Code +     │                      │ Static Analysis +     │
   │ MCP Servers       │                      │ Ghidra                │
   │ NAT + Host-only   │                      │ Host-only · .56.20    │
   │ .56.10            │                      └──────────────────────┘
   │                   │                      ┌──────────────────────┐
   │                   │ ───────────────────► │ VM3                   │
   │                   │                      │ SIFT Workstation       │
   │                   │                      │ Disk / Memory / Timeline│
   │                   │                      │ Host-only · .56.30    │
   │                   │                      └──────────────────────┘
   │                   │                      ┌──────────────────────┐
   │                   │ ───────────────────► │ VM4                   │
   │                   │                      │ Windows 10             │
   │                   │                      │ Dynamic Analysis +     │
   │                   │                      │ x64dbg                 │
   └──────────────────┘                      │ Host-only · .56.40     │
                                              │ (future)               │
                                              └──────────────────────┘
```

> **ℹ️ Key principle:** VM1 (Ubuntu) is the only machine with internet access. Samples are transferred to REMnux or SIFT via SCP and never executed on VM1. Claude orchestrates all analysis remotely over SSH.

| VM | OS | Role | IP | Internet |
|----|----|------|----|----------|
| `VM1` | Ubuntu 24.04 LTS | Claude Code + MCP orchestration host | `192.168.56.10` | ✅ Yes |
| `VM2` | REMnux (Ubuntu-based) | Static analysis, Ghidra, deobfuscation, malware triage | `192.168.56.20` | ❌ No |
| `VM3` | SIFT Workstation (Ubuntu 22.04) | Disk forensics, memory analysis, timeline, registry | `192.168.56.30` | ❌ No |
| `VM4` | Windows 10 | Dynamic analysis, x64dbg debugging, FLARE-VM | `192.168.56.40` | 🔶 Future |

---

## 02. Prerequisites

Download everything before configuring the network. All VMs need internet to install packages — complete all installs before switching to host-only.

> **⚠️ Important:** Keep all VMs on **NAT** during installation. Only switch each VM to host-only after all its tools are installed and verified.

### Host Machine Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 16 GB | 32 GB |
| Storage | 150 GB free | 300 GB SSD |
| CPU | 4 cores | 8 cores with VT-x |
| Hypervisor | VMware Workstation Pro / Fusion | VMware Workstation Pro / Fusion |

### Downloads Required

- [ ] **Ubuntu 24.04 LTS ISO** — [ubuntu.com/download](https://ubuntu.com/download/desktop) — for VM1
- [ ] **REMnux OVA** — [remnux.org](https://remnux.org) — pre-built analysis VM
- [ ] **SIFT Workstation OVA** — [sans.org/tools/sift-workstation](https://www.sans.org/tools/sift-workstation) — requires free SANS account. Default login: `sansforensics` / `forensics`
- [ ] **VMware Workstation / Fusion** — [vmware.com](https://www.vmware.com)
- [ ] **Anthropic account** with Claude Pro, Max, Team, or Enterprise subscription
- [ ] **VirusTotal API key** — Premium recommended for file downloads — [virustotal.com](https://www.virustotal.com)

### VM Allocation

| VM | RAM | vCPU | Disk | Status |
|----|-----|------|------|--------|
| VM1 — Ubuntu 24.04 | 4 GB | 2 | 40 GB | ✅ Active |
| VM2 — REMnux | 4 GB | 2 | 60 GB | ✅ Active |
| VM3 — SIFT Workstation | 4 GB | 2 | 80 GB | ✅ Active |
| VM4 — Windows 10 | 4 GB | 2 | 80 GB | 🔶 Future |

---

## 03. VM1 — Ubuntu 24.04 (LLM Host)

Install Ubuntu 24.04, then install Claude Code and all MCP server dependencies. Keep VM1 on NAT throughout this section.

> **ℹ️** VM1 network adapter: set to **NAT** for now. You will add a second host-only adapter in the Network Configuration section.

### Step 1 — System Update

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git build-essential python3 python3-pip zip unzip
```

### Step 2 — Install Claude Code

```bash
# Install Claude Code via native installer
curl -fsSL https://claude.ai/install.sh | bash

# Verify
claude --version
claude doctor
```

### Step 3 — Authenticate Claude Code

```bash
# Launch Claude — follow the device code prompt
claude
# 1. Select "Sign in with Claude.ai"
# 2. Open the displayed URL in your browser
# 3. Enter the device code and sign in
```

### Step 4 — Install Node.js 20

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node --version   # v20.x.x
npm --version
```

### Step 5 — Store API Keys Securely

```bash
mkdir -p ~/.config/malware-lab
cat > ~/.config/malware-lab/secrets.env << 'EOF'
VIRUSTOTAL_API_KEY=paste_your_vt_key_here
EOF
chmod 600 ~/.config/malware-lab/secrets.env
echo 'source ~/.config/malware-lab/secrets.env' >> ~/.bashrc
source ~/.bashrc
```

### Step 6 — Create Working Directory and CLAUDE.md

```bash
mkdir -p ~/malware-analysis ~/downloads

cat > ~/malware-analysis/CLAUDE.md << 'EOF'
# Malware Analysis Lab

## Startup
At the start of every session, list all available tools grouped by category:
1. remnux-ssh MCP tools grouped by [CATEGORY] tag
2. sift-ssh MCP tools grouped by [CATEGORY] tag
3. virustotal MCP tools with a short description of each
Then wait for the next instruction.

## VMs
- VM1 Ubuntu:  192.168.56.10 — this machine (Claude Code + MCP host)
- VM2 REMnux:  192.168.56.20 — static analysis, Ghidra
- VM3 SIFT:    192.168.56.30 — disk/memory/timeline forensics

## SSH Keys & Commands
- ~/.ssh/remnux_key → VM2: ssh -i ~/.ssh/remnux_key remnux@192.168.56.20
- ~/.ssh/sift_key   → VM3: ssh -i ~/.ssh/sift_key sansforensics@192.168.56.30

## MCP Servers
- remnux-ssh  → static analysis tools on REMnux
- sift-ssh    → forensic tools on SIFT
- virustotal  → hash lookups, detections, sample downloads (password: infected)
- ghidra-mcp  → NOT used directly; use remnux-ssh ghidra_* tools instead

## Sample Transfer
scp -i ~/.ssh/remnux_key <file> remnux@192.168.56.20:/home/remnux/intake/
scp -i ~/.ssh/sift_key <file> sansforensics@192.168.56.30:/cases/

## Directories — REMnux (VM2)
- Intake:   /home/remnux/intake/
- Samples:  /home/remnux/samples/
- Ghidra container maps /home/remnux/samples/ → /data/

## Directories — SIFT (VM3)
- Evidence: /cases/   Memory: /cases/*.dmp   Disk: /cases/*.dd

## Malware Analysis Workflow (REMnux)
1. Download via virustotal MCP → ~/downloads/sample.zip
2. Transfer: scp -i ~/.ssh/remnux_key ~/downloads/sample.zip remnux@192.168.56.20:/home/remnux/intake/
3. Unzip: remnux-ssh intake_sample (default password: infected)
4. Triage: triage_* tool based on file type
5. Deep analysis: ghidra_load_sample → ghidra_* tools
6. Intelligence: virustotal get_file_report

## Forensic Workflow (SIFT)
1. Transfer evidence: scp -i ~/.ssh/sift_key <evidence> sansforensics@192.168.56.30:/cases/
2. Hash: sift-ssh hash_file
3. Memory: sift-ssh triage_memory
4. Disk: sift-ssh triage_disk
5. Timeline: timeline_create → timeline_filter → timeline_search
6. Registry: registry_analyze

## Ghidra Headless Server (via remnux-ssh)
- Auth: Authorization: Bearer ghidra-lab-token
- Load PE/ELF: ghidra_load_sample with path only (auto-detects)
- Load shellcode/raw: ghidra_load_sample with path + language
  - x86 64-bit:  x86:LE:64:default
  - x86 32-bit:  x86:LE:32:default
  - ARM 32-bit:  ARM:LE:32:v8
  - ARM 64-bit:  AARCH64:LE:64:v8A
- After raw blobs: call ghidra_run_analysis with entry_offset=0x0
- Workflow: ghidra_load_sample → ghidra_run_analysis → ghidra_list_functions → ghidra_decompile
- Path mapping: /home/remnux/samples/ → /data/ (handled automatically by ghidra-post.sh)

## Rules
- Never store or execute samples on VM1
- Samples only live in /home/remnux/samples/ on REMnux
- Evidence only in /cases/ on SIFT
- Always revert REMnux snapshot after analysis
EOF
```

### Step 7 — Create the `lab` Shortcut

Adds a shell function that starts Claude with all tools listed at startup.

```bash
cat >> ~/.bashrc << 'EOF'

lab() {
  cd ~/malware-analysis
  echo "Loading malware analysis lab..."
  claude "Show all available tools for this malware analysis lab grouped by category:
1. List all tools from remnux-ssh MCP grouped by their [CATEGORY] tag
2. List all tools from sift-ssh MCP grouped by their [CATEGORY] tag
3. List all tools from virustotal MCP with a short description of each
After showing the full list, wait for my next instruction."
}
EOF
source ~/.bashrc
```

> **✅** Type `lab` from anywhere to start an analysis session. Claude will list all tools then wait for your first prompt.

---

## 04. VM2 — REMnux (Analysis Machine)

Import the REMnux OVA, install all tools while NAT is active, then switch to host-only. Complete all steps before changing the network adapter.

> **⚠️** Keep REMnux on **NAT** during this entire section.

### Step 1 — Import REMnux OVA

In VMware: **File → Open** → select the REMnux `.ova` → Import. Allocate 4 GB RAM, 2 vCPUs.

### Step 2 — Install SSH Server

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable ssh && sudo systemctl start ssh
sudo ss -tlnp | grep 22  # verify listening
```

### Step 3 — Install Additional Analysis Tools

```bash
# Python tools
pip3 install vipermonkey uncompyle6 dnfile pefile blint --break-system-packages

# JavaScript deobfuscation
npm install -g synchrony js-beautify

# Go runtime (for Go binary analysis)
sudo apt install -y golang-go

# Verify key tools
which olevba oleid mraptor diec floss capa
which readelf objdump nm binwalk exiftool
which js-beautify synchrony cfr procyon javap
/usr/local/jadx/bin/jadx --version
go version
blint --version 2>/dev/null || echo "blint installed as Python package"
```

> **ℹ️** Most tools are pre-installed on REMnux. The commands above install the few that require manual installation. Run them all — existing installs will simply be skipped.

### Step 4 — Install Docker (for Ghidra MCP)

```bash
docker --version  # Docker is typically pre-installed on REMnux
# If not installed:
sudo apt install -y docker.io
sudo systemctl enable docker
sudo usermod -aG docker $USER && newgrp docker
```

### Step 5 — Create Sample Directories

```bash
mkdir -p /home/remnux/samples /home/remnux/intake /home/remnux/ghidra-scripts
```

---

## 05. VM3 — SIFT Workstation (Forensics)

Import the SIFT OVA, install additional tools while NAT is active, configure static IP, then switch to host-only. Default credentials: `sansforensics` / `forensics`.

> **⚠️** Keep SIFT on **NAT** during this entire section.

### Step 1 — Enable Copy/Paste from Host

In VMware: **VM → Settings → Options → Guest Isolation** → enable *"Allow copy and paste"*. Then:

```bash
sudo apt install -y open-vm-tools open-vm-tools-desktop
sudo systemctl restart open-vm-tools
sudo reboot
```

### Step 2 — Install SSH Server

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable ssh && sudo systemctl start ssh
sudo ss -tlnp | grep 22
```

### Step 3 — Install Additional Forensic Tools

```bash
sudo apt update
sudo apt install -y sleuthkit yara python3-yara \
  dcfldd hashdeep ssdeep exiftool \
  libewf2t64 afflib-tools git python3-pip

# Python forensic libraries
pip3 install python-registry construct dissect.cstruct \
  --break-system-packages 2>&1 | tail -3

# Verify key tools
vol -h 2>/dev/null | head -2  # Volatility3
which fls icat fsstat mmls    # Sleuth Kit
which log2timeline.py psteal.py  # Plaso
which rip.pl bulk_extractor foremost dcfldd ssdeep yara
```

### Step 4 — Create Cases Directory

```bash
sudo mkdir -p /cases /cases/carved /cases/dumps /cases/recovered
sudo chown -R sansforensics:sansforensics /cases
```

### Step 5 — Set Static IP

```bash
# Check interface name first
ip a | grep -E "^[0-9]+:" | grep -v lo

# Write netplan config (use tee — nano not installed on SIFT)
sudo tee /etc/netplan/01-network-manager-all.yaml << 'EOF'
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.56.30/24]
EOF

sudo netplan apply
ip a | grep 192.168.56.30  # verify
```

> **ℹ️** After applying the static IP, switch the VMware network adapter to **Host-only (VMnet1)**. SIFT will then be reachable from VM1 at `192.168.56.30`.

---

## 06. Network Configuration

Configure VMware adapters and static IPs on VM1 and VM2. Do this after all tools are installed.

### VMware Adapter Configuration

| VM | Adapter 1 | Adapter 2 |
|----|-----------|-----------|
| VM1 — Ubuntu | NAT — internet access | Host-only VMnet1 — lab network |
| VM2 — REMnux | Host-only VMnet1 — lab network only | — |
| VM3 — SIFT | Host-only VMnet1 — lab network only | — |

### Static IP on VM1 (Ubuntu)

```bash
# Find interface name (look for second adapter, not NAT one)
ip a
sudo nano /etc/netplan/01-network-manager-all.yaml
```

`/etc/netplan/01-network-manager-all.yaml` (VM1):

```yaml
network:
  version: 2
  ethernets:
    ens33:           # NAT adapter — keep as DHCP
      dhcp4: true
    ens36:           # Host-only adapter — static IP
      dhcp4: no
      addresses: [192.168.56.10/24]
```

```bash
sudo netplan apply
```

### Static IP on VM2 (REMnux)

`/etc/netplan/01-network-manager-all.yaml` (VM2):

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.56.20/24]
```

```bash
sudo netplan apply
ping -c 3 192.168.56.10  # test VM1 reachability
```

> **✅** VM1, VM2 and VM3 should all be able to ping each other on `192.168.56.0/24`. VM1 should also have internet access via NAT.

---

## 07. SSH Key Setup

Generate SSH keys on VM1 for both REMnux and SIFT. Required for MCP servers to communicate.

**1. Generate keys on VM1**

```bash
ssh-keygen -t ed25519 -f ~/.ssh/remnux_key -N "" -C "vm1-to-remnux"
ssh-keygen -t ed25519 -f ~/.ssh/sift_key   -N "" -C "vm1-to-sift"
```

**2. Copy keys to VMs**

```bash
ssh-copy-id -i ~/.ssh/remnux_key.pub remnux@192.168.56.20
ssh-copy-id -i ~/.ssh/sift_key.pub   sansforensics@192.168.56.30
```

**3. Create SSH config** (`~/.ssh/config` on VM1)

```
Host remnux
    HostName 192.168.56.20
    User remnux
    IdentityFile ~/.ssh/remnux_key
    StrictHostKeyChecking no

Host sift
    HostName 192.168.56.30
    User sansforensics
    IdentityFile ~/.ssh/sift_key
    StrictHostKeyChecking no
```

**4. Test connectivity**

```bash
ssh remnux "uname -a && echo REMnux OK"
ssh sift "uname -a && vol -h 2>/dev/null | head -2 && echo SIFT OK"
```

---

## 08. remnux-ssh MCP Server

Build and register the custom MCP server that lets Claude run commands on REMnux over SSH.

### Build the MCP Server

```bash
mkdir -p ~/mcp/remnux-mcp/src && cd ~/mcp/remnux-mcp
npm init -y
npm install @modelcontextprotocol/sdk ssh2 zod
npm install -D typescript @types/node @types/ssh2
npm pkg set type="module"

cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2022", "module": "NodeNext",
    "moduleResolution": "NodeNext", "outDir": "build",
    "rootDir": "src", "strict": false,
    "esModuleInterop": true, "skipLibCheck": true
  },
  "include": ["src"]
}
EOF

# Place src/index.ts then build
npx tsc && ls build/
```

### Create Ghidra Helper Scripts on REMnux

These scripts solve SSH quoting issues when communicating with the Ghidra Docker container. The POST script automatically translates REMnux paths to container paths and supports an optional language parameter for shellcode/raw binaries.

```bash
ssh remnux 'python3 -c "
import os
os.makedirs(\"/home/remnux/ghidra-scripts\", exist_ok=True)

get_script = \"\"\"#!/bin/sh
echo \"curl -s -H \x27Authorization: Bearer ghidra-lab-token\x27 http://localhost:8089\$1\" > /tmp/gh_cmd.sh
docker cp /tmp/gh_cmd.sh ghidra-mcp:/tmp/gh_cmd.sh
docker exec ghidra-mcp sh /tmp/gh_cmd.sh
\"\"\"

post_script = \"\"\"#!/bin/sh
# Translate REMnux path to Docker container path automatically
FILE_PATH=\$(echo \"\$1\" | sed \x27s|/home/remnux/samples/|/data/|\x27)
LANGUAGE=\$2
if [ -n \"\$LANGUAGE\" ]; then
  BODY=\"{\\\"file\\\":\\\"\$FILE_PATH\\\",\\\"language\\\":\\\"\$LANGUAGE\\\"}\"
else
  BODY=\"{\\\"file\\\":\\\"\$FILE_PATH\\\"}\"
fi
echo \"\$BODY\" > /tmp/gh_body.json
echo \"curl -s -X POST -H \x27Authorization: Bearer ghidra-lab-token\x27 -H \x27Content-Type: application/json\x27 -d @/tmp/gh_body.json http://localhost:8089/load_program\" > /tmp/gh_post_cmd.sh
docker cp /tmp/gh_body.json ghidra-mcp:/tmp/gh_body.json
docker cp /tmp/gh_post_cmd.sh ghidra-mcp:/tmp/gh_post_cmd.sh
docker exec ghidra-mcp sh /tmp/gh_post_cmd.sh
\"\"\"

with open(\"/home/remnux/ghidra-scripts/ghidra-get.sh\", \"w\") as f: f.write(get_script)
with open(\"/home/remnux/ghidra-scripts/ghidra-post.sh\", \"w\") as f: f.write(post_script)
os.chmod(\"/home/remnux/ghidra-scripts/ghidra-get.sh\", 0o755)
os.chmod(\"/home/remnux/ghidra-scripts/ghidra-post.sh\", 0o755)
print(\"Scripts written\")
"'
```

> **ℹ️** The POST script maps `/home/remnux/samples/` → `/data/` automatically. Pass either the REMnux path or the container path — both work. For raw shellcode, pass the language as a second argument, e.g. `x86:LE:64:default`.

### Register with Claude Code

```bash
claude mcp add remnux-ssh \
  --scope user \
  -- node /home/$USER/mcp/remnux-mcp/build/index.js \
  --host 192.168.56.20 \
  --username remnux \
  --private-key /home/$USER/.ssh/remnux_key

claude mcp list | grep remnux-ssh
```

---

## 09. VirusTotal MCP Server

Set up the VirusTotal MCP for hash lookups, file reports, and sample downloads.

> **ℹ️** File download requires a **VirusTotal Premium API key**. Hash lookups and reports work with a free key.

```bash
git clone https://github.com/BurtTheCoder/mcp-virustotal ~/mcp/mcp-virustotal
cd ~/mcp/mcp-virustotal && npm install && npm run build

source ~/.config/malware-lab/secrets.env
claude mcp add virustotal \
  --scope user \
  -e VIRUSTOTAL_API_KEY=$VIRUSTOTAL_API_KEY \
  -- node /home/$USER/mcp/mcp-virustotal/build/index.js
```

### VirusTotal MCP — Available Tools

| Tool | Input | Description | Tier |
|------|-------|-------------|------|
| `get_file_report` | MD5/SHA1/SHA256 | Full detection report — AV verdicts, sandbox scores, metadata | Free |
| `get_file_behavior` | SHA256 | Dynamic analysis — network connections, registry changes, dropped files | Free |
| `get_file_relationship` | SHA256 + type | Related files, contacted IPs/domains, execution parents | Free |
| `get_url_report` | URL | URL reputation, category, detections | Free |
| `get_ip_report` | IP address | IP reputation, ASN, country, associated malware | Free |
| `get_domain_report` | Domain | Domain reputation, WHOIS, DNS history | Free |
| `search` | VTI query | Advanced search with VTI modifiers, e.g. `type:peexe tag:signed` | Free |
| `download_file` | SHA256 + path | Download sample as password-protected zip (password: `infected`) | Premium |

> **ℹ️** `download_file` saves to `~/downloads/` on VM1. Always transfer to REMnux via `scp` before unzipping — never extract samples on VM1.

---

## 10. Ghidra MCP (Docker Headless)

Run GhidraMCP as a headless Docker container on REMnux. Build the image on VM1 (internet access) and transfer to REMnux.

### Step 1 — Build Docker Image on VM1

```bash
sudo apt install -y docker.io
sudo systemctl start docker
sudo usermod -aG docker $USER && newgrp docker

git clone https://github.com/bethington/ghidra-mcp ~/mcp/ghidra-mcp
cd ~/mcp/ghidra-mcp
docker build -t ghidra-mcp-headless:latest -f docker/Dockerfile .

docker save ghidra-mcp-headless:latest -o /tmp/ghidra-mcp-headless.tar
ls -lh /tmp/ghidra-mcp-headless.tar
```

### Step 2 — Transfer and Load on REMnux

```bash
scp -i ~/.ssh/remnux_key /tmp/ghidra-mcp-headless.tar remnux@192.168.56.20:/home/remnux/
```

```bash
docker load -i /home/remnux/ghidra-mcp-headless.tar

python3 -c "
import os
script = '''#!/bin/bash
docker run -d --name ghidra-mcp --memory 2g \
  -p 127.0.0.1:8089:8089 \
  -v /home/remnux/samples:/data \
  -v ghidra-projects:/projects \
  -e GHIDRA_MCP_PORT=8089 \
  -e GHIDRA_MCP_BIND_ADDRESS=0.0.0.0 \
  -e GHIDRA_MCP_AUTH_TOKEN=ghidra-lab-token \
  -e JAVA_OPTS=-Xmx2g \
  ghidra-mcp-headless:latest
'''
with open('/home/remnux/run-ghidra-mcp.sh', 'w') as f: f.write(script)
os.chmod('/home/remnux/run-ghidra-mcp.sh', 0o755)
"

/home/remnux/run-ghidra-mcp.sh
sleep 20
docker exec ghidra-mcp curl -s http://localhost:8089/get_version
```

> **✅** Expected: `{"plugin_version": "5.13.1-headless", "mode": "headless"}`

---

## 11. sift-ssh MCP Server

Build and register the sift-ssh MCP server — same pattern as remnux-ssh.

```bash
mkdir -p ~/mcp/sift-mcp/src && cd ~/mcp/sift-mcp
npm init -y
npm install @modelcontextprotocol/sdk ssh2 zod
npm install -D typescript @types/node @types/ssh2
npm pkg set type="module"
# Copy tsconfig.json from remnux-mcp and update src/index.ts
npx tsc

claude mcp add sift-ssh \
  --scope user \
  -- node /home/$USER/mcp/sift-mcp/build/index.js \
  --host 192.168.56.30 \
  --username sansforensics \
  --private-key /home/$USER/.ssh/sift_key

# Test
claude mcp list | grep sift
```

> **ℹ️** Test with: *"Use sift-ssh run_command to run: uname -a && vol -h 2>/dev/null | head -2"*

---

## 12. REMnux Analysis Tools

Complete tool inventory available on REMnux, organized by category. All tools tagged `install` require manual installation (Step 3 of VM2 setup).

### Static Analysis

| Tool | Description | Source |
|------|-------------|--------|
| `file` | Identify file type by magic bytes | built-in |
| `strings` | Extract printable strings | built-in |
| `xxd` | Hex dump files | built-in |
| `diec` / `die` | Detect packers, compilers, protectors | pre-installed |
| `FLOSS` | Extract obfuscated + stack strings | pre-installed |
| `CAPA` | MITRE ATT&CK capability detection | pre-installed |
| `pescanner` | PE header and section analysis | pre-installed |
| `exiftool` | File metadata extraction | pre-installed |
| `binwalk` | Embedded file extraction | pre-installed |
| `upx` | UPX packer detection / unpacking | pre-installed |

### Office Document Analysis

| Tool | Description | Source |
|------|-------------|--------|
| `olevba` | VBA macro extraction and analysis | pre-installed |
| `oleid` | OLE file indicator detection | pre-installed |
| `oledump` | OLE stream dumping | pre-installed |
| `mraptor` | Malicious macro detection | pre-installed |
| `ViperMonkey` | VBA macro emulation | install |

### Script Deobfuscation

| Tool | Description | Source |
|------|-------------|--------|
| `synchrony` | JavaScript deobfuscation (obfuscator.io) | install |
| `js-beautify` | JavaScript formatting and beautification | install |

### ELF / Linux Analysis

| Tool | Description | Source |
|------|-------------|--------|
| `readelf` | ELF header and section analysis | built-in |
| `objdump` | Disassembly and symbol inspection | built-in |
| `nm` | Symbol table listing | built-in |
| `ldd` | Shared library dependencies | built-in |
| `strace` / `ltrace` | System and library call tracing | pre-installed |

### Python Malware

| Tool | Description | Source |
|------|-------------|--------|
| `pyinstxtractor-ng` | Unpack PyInstaller bundles | pre-installed |
| `uncompyle6` | Decompile Python .pyc bytecode | install |

### Windows Installer (MSI / MSIX)

| Tool | Description | Source |
|------|-------------|--------|
| `msiextract` | Extract MSI file contents | pre-installed |
| `msidump` | Dump MSI database tables | pre-installed |
| `7z` / `unzip` | Extract MSIX/AppX packages | pre-installed |
| `cabextract` | Extract Cabinet files | pre-installed |

### .NET Assembly Analysis

| Tool | Description | Source |
|------|-------------|--------|
| `ilspycmd` | Decompile .NET assemblies to C# source | pre-installed |
| `monodis` | Disassemble .NET IL bytecode, extract user strings | pre-installed |
| `pedump` | PE/CLI header and metadata table analysis | pre-installed |
| `dnfile` | Python library for .NET metadata parsing | install |
| `pefile` | Python PE structure parsing library | install |
| `dotnet 8.0 runtime` | Required by ilspycmd | pre-installed |

### Java / JAR Analysis

| Tool | Description | Source |
|------|-------------|--------|
| `cfr` | Decompile JAR/class to Java source — handles modern Java well | pre-installed |
| `procyon` | Alternative Java decompiler — better with obfuscated code | pre-installed |
| `javap` | Disassemble .class bytecode to JVM instructions | built-in |
| `jadx` | Decompile JAR and Android APK — best for obfuscated JARs | pre-installed |
| `java` / `jar` | JDK tools for manifest inspection and class listing | built-in |

### Go Binary Analysis

| Tool | Description | Source |
|------|-------------|--------|
| `go version -m` | Extract build info, module dependencies from Go binary | install |
| `blint` | Go binary capability detection — MITRE ATT&CK mapping | install |
| `nm` / `objdump` | Symbol extraction — Go symbols are rarely stripped | built-in |
| `strings` | Go binaries embed module paths unobfuscated — very effective | built-in |

---

## 13. SIFT Forensic Tools

Complete tool inventory available on SIFT Workstation. Most are pre-installed; a few require manual installation (Step 3 of VM3 setup).

### Memory Forensics

| Tool | Description | Source |
|------|-------------|--------|
| `Volatility3 (vol)` | Windows/Linux/macOS memory analysis — pslist, netscan, malfind, cmdline, dlllist | pre-installed |

### Disk Forensics

| Tool | Description | Source |
|------|-------------|--------|
| `fls` / `icat` | List and extract files from disk images | pre-installed |
| `fsstat` / `mmls` | Filesystem and partition info | pre-installed |
| `tsk_recover` | Recover files from disk images | pre-installed |
| `Autopsy` | GUI forensic browser for disk images | pre-installed |

### Timeline Analysis

| Tool | Description | Source |
|------|-------------|--------|
| `log2timeline.py` | Create super-timelines from disk images and artifacts | pre-installed |
| `psteal.py` / `psort.py` | Filter, sort and export plaso timeline events | pre-installed |

### Artifact Extraction

| Tool | Description | Source |
|------|-------------|--------|
| `bulk_extractor` | Extract emails, URLs, credit cards, domains from images | pre-installed |
| `foremost` | Carve deleted files from disk images | pre-installed |
| `dcfldd` | Forensic disk copy with hash verification | install |
| `ssdeep` | Fuzzy hashing for file similarity | install |

### Registry Forensics

| Tool | Description | Source |
|------|-------------|--------|
| `RegRipper (rip.pl)` | Extract autoruns, user activity, program execution from registry hives | pre-installed |

---

## 14. MCP Tool Groups

Tools are organized into groups via tags in their descriptions. Claude uses these tags to pick the right tool automatically.

### remnux-ssh MCP Tool Groups

| Group | Tag | Key Tools | Use when |
|-------|-----|-----------|----------|
| Intake & Utility | `[UTILITY]` | `run_command`, `upload_sample`, `intake_sample`, `list_samples` | Transferring and managing samples |
| Quick Triage | `[TRIAGE]` | `triage_quick`, `triage_pe`, `triage_iocs`, `triage_office`, `triage_script`, `triage_elf`, `triage_python`, `triage_java`, `triage_go`, `triage_dotnet`, `triage_full` | First look at any new sample |
| Deep Static | `[STATIC]` | `analyze_file`, `floss_strings`, `capa_analyze`, `diec_analyze`, `binwalk_extract`, `extract_config` | Detailed analysis after triage |
| Office Documents | `[OFFICE]` | `oleid_analyze`, `olevba_analyze`, `oledump_analyze`, `mraptor_analyze`, `vmonkey_analyze` | doc/docx/xls/xlsx/ppt |
| Script Deobfuscation | `[DEOBFUSCATE]` | `deobfuscate_js`, `js_beautify`, `js_extract_strings`, `deobfuscate_powershell` | JS/PS1/VBS/BAT scripts |
| MSI / MSIX | `[MSI]` | `msi_analyze`, `msi_extract_files`, `msix_analyze` | Windows installer packages |
| ELF / Linux | `[ELF]` | `elf_analyze`, `elf_imports`, `elf_sections`, `elf_iocs` | Linux ELF binaries |
| Python Malware | `[PYTHON]` | `pyinstaller_extract`, `python_decompile`, `python_analyze` | PyInstaller bundles, .pyc files |
| .NET Assembly | `[DOTNET]` | `dotnet_decompile`, `dotnet_il_disasm`, `dotnet_pe_headers`, `dotnet_metadata`, `dotnet_strings`, `dotnet_resources` | C#, VB.NET, F# assemblies |
| Java / JAR | `[JAVA]` | `java_decompile`, `java_disassemble`, `java_jar_inspect`, `java_extract_jar`, `jadx_decompile`, `java_strings_iocs` | JAR files, .class, Android APKs |
| Go Binaries | `[GO]` | `go_info`, `go_symbols`, `go_strings`, `go_capabilities`, `go_modules` | Compiled Go binaries |
| Ghidra | `[GHIDRA]` | `ghidra_load_sample`, `ghidra_run_analysis`, `ghidra_list_functions`, `ghidra_decompile`, `ghidra_list_imports`, `ghidra_list_strings`, `ghidra_detect_malware`, `ghidra_extract_iocs` | Deep reverse engineering — call `ghidra_run_analysis` after loading shellcode/raw binaries |

### sift-ssh MCP Tool Groups

| Group | Tag | Key Tools | Use when |
|-------|-----|-----------|----------|
| Utility | `[UTILITY]` | `run_command`, `list_cases`, `hash_file` | Case management, evidence hashing |
| Memory Forensics | `[MEMORY]` | `memory_info`, `memory_pslist`, `memory_netscan`, `memory_malfind`, `memory_cmdline`, `memory_dlllist`, `memory_handles`, `memory_registry`, `memory_dump_process`, `triage_memory` | Memory dumps (.dmp, .raw) |
| Disk Forensics | `[DISK]` | `disk_info`, `disk_list_files`, `disk_recover_files`, `disk_extract_file`, `triage_disk` | Disk images (.dd, .E01) |
| Timeline Analysis | `[TIMELINE]` | `timeline_create`, `timeline_filter`, `timeline_search` | Super-timeline creation |
| Artifact Extraction | `[ARTIFACT]` | `extract_artifacts`, `carve_files`, `forensic_copy` | Bulk extraction, file carving |
| Registry Forensics | `[REGISTRY]` | `registry_analyze`, `registry_autoruns`, `registry_useractivity` | Windows registry hive analysis |
| Triage | `[TRIAGE]` | `triage_case`, `triage_memory`, `triage_disk` | First look at any evidence file |

---

## 15. Analysis Workflows

Standard prompts for Claude Code. Run from `~/malware-analysis` using the `lab` command.

### Workflow 1 — Full PE Sample Analysis

```
Full malware analysis workflow for SHA256 <HASH>:
1. Use virustotal download_file to save to ~/downloads/sample.zip
2. scp to /home/remnux/intake/sample.zip on REMnux
3. remnux-ssh intake_sample (password: infected)
4. remnux-ssh triage_full on the extracted sample
5. virustotal get_file_report for the hash
6. remnux-ssh ghidra_load_sample (path only — auto-detects PE/ELF)
7. remnux-ssh ghidra_list_functions then ghidra_list_imports
8. remnux-ssh ghidra_decompile on any suspicious functions found
9. Write triage report: file type, hashes, capabilities, IOCs, VT detections
```

### Workflow 1b — Shellcode / Raw Binary Analysis

```
Analyze the raw shellcode/headerless binary at /home/remnux/samples/shellcode.bin:
1. remnux-ssh triage_quick for file type and hashes
2. remnux-ssh ghidra_load_sample with:
   - path: /home/remnux/samples/shellcode.bin
   - language: x86:LE:64:default  (or x86:LE:32:default for 32-bit)
3. remnux-ssh ghidra_run_analysis with entry_offset=0x0
4. remnux-ssh ghidra_list_functions to see discovered functions
5. remnux-ssh ghidra_decompile on each function found
6. Summarize: shellcode purpose, API calls, IOCs, likely malware family

Available language IDs:
- x86 64-bit:  x86:LE:64:default
- x86 32-bit:  x86:LE:32:default
- ARM 32-bit:  ARM:LE:32:v8
- ARM 64-bit:  AARCH64:LE:64:v8A
```

### Workflow 2 — Decompile Specific Functions

```
After loading a sample into Ghidra, decompile specific functions:

# By function name:
Use remnux-ssh ghidra_decompile on function named FUN_140001000

# By address:
Use remnux-ssh ghidra_decompile on function at address 0x140001000

# Batch decompile suspicious functions:
Use remnux-ssh to:
1. Call ghidra_list_functions
2. Identify functions with suspicious names (inject, decrypt, download, exec, socket, connect)
3. Decompile each suspicious function using ghidra_decompile
4. Summarize the capabilities found
```

### Workflow 2 — Office Document Analysis

```
Analyze the Office document at /home/remnux/samples/invoice.docx:
1. remnux-ssh triage_office (runs all oletools)
2. remnux-ssh vmonkey_analyze to emulate any macros
3. Extract all IOCs (URLs, IPs, domains)
4. Summarize: macro presence, auto-exec triggers, IOCs, verdict
```

### Workflow 3 — JavaScript Deobfuscation

```
Deobfuscate /home/remnux/samples/malicious.js:
1. remnux-ssh triage_script
2. remnux-ssh deobfuscate_js with method=both
3. Extract all network IOCs from the deobfuscated output
4. Identify malware family if possible
```

### Workflow 4 — PyInstaller Sample

```
Analyze the Python binary at /home/remnux/samples/stealer.exe:
1. remnux-ssh triage_python
2. If PyInstaller detected, remnux-ssh pyinstaller_extract
3. remnux-ssh python_decompile on any .pyc files found
4. Extract C2 configuration from decompiled source
5. Cross-reference IOCs with virustotal
```

### Workflow 5 — .NET Assembly Analysis

```
Analyze the .NET sample at /home/remnux/samples/agent.exe:
1. remnux-ssh triage_dotnet
2. remnux-ssh dotnet_decompile for full C# source
3. remnux-ssh dotnet_strings for obfuscated strings
4. remnux-ssh dotnet_resources for embedded payloads
5. virustotal get_file_report
6. Summarize: obfuscator, capabilities, IOCs, verdict
```

### Workflow 6 — Java / JAR Analysis

```
Analyze the JAR at /home/remnux/samples/payload.jar:
1. remnux-ssh triage_java
2. remnux-ssh java_jar_inspect for class and resource listing
3. remnux-ssh java_decompile with tool=both (CFR + Procyon)
4. If obfuscated, remnux-ssh jadx_decompile
5. remnux-ssh java_strings_iocs for C2 indicators
6. virustotal get_file_report
7. Summarize: functionality, obfuscation, IOCs, malware family
```

### Workflow 7 — Go Binary Analysis

```
Analyze the Go binary at /home/remnux/samples/implant:
1. remnux-ssh triage_go
2. remnux-ssh go_info for build version and metadata
3. remnux-ssh go_modules for third-party libraries
4. remnux-ssh go_symbols for exported functions
5. remnux-ssh go_capabilities for MITRE ATT&CK mapping
6. remnux-ssh go_strings for network IOCs
7. Cross-reference IOCs with virustotal
8. Summarize: C2 framework, capabilities, persistence, IOCs
```

### Workflow 8 — Memory Forensics

```
Analyze the memory dump at /cases/memory.dmp on SIFT:
1. sift-ssh triage_memory
2. Identify suspicious processes from pslist
3. sift-ssh memory_malfind for injected code
4. For suspicious PIDs: memory_cmdline and memory_dlllist
5. sift-ssh memory_netscan for C2 connections
6. Cross-reference IPs/domains with virustotal
7. sift-ssh memory_dump_process to extract suspicious process
8. Transfer to REMnux for static analysis
9. Summarize: compromised processes, IOCs, attacker activity
```

### Workflow 9 — Disk Image Forensics

```
Investigate the disk image at /cases/disk.dd on SIFT:
1. sift-ssh hash_file to verify evidence integrity
2. sift-ssh triage_disk for partition and filesystem info
3. sift-ssh disk_list_files to browse the filesystem
4. sift-ssh timeline_create to build a super-timeline
5. sift-ssh timeline_search around the incident timeframe
6. sift-ssh carve_files to recover deleted executables
7. sift-ssh extract_artifacts for bulk IOC extraction
8. Transfer carved executables to REMnux for static analysis
9. Summarize: timeline of events, suspicious files, IOCs
```

---

## 16. Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| SSH connection refused | SSH service not running or wrong IP | Run `sudo systemctl start ssh`. Verify IP with `ip a`. |
| MCP shows "Failed to connect" | Node.js path wrong or build not compiled | Run `npx tsc`. Re-register with `claude mcp remove` then `add`. |
| Ghidra returns "Unauthorized" | Auth token stripped through SSH layers | Use the ghidra-scripts wrapper approach — never pass token via `docker exec` args directly. |
| Ghidra container crashes | Binding 0.0.0.0 without auth token | Always set `GHIDRA_MCP_AUTH_TOKEN` when using `GHIDRA_MCP_BIND_ADDRESS=0.0.0.0`. |
| `ghidra_list_functions` returns empty | Sample loaded but not analyzed | Call `ghidra_run_analysis` with `entry_offset=0x0` after loading. Required for raw shellcode and headerless binaries. |
| `ghidra_load_sample` fails on shellcode | Auto-detection fails on headerless binaries | Pass the `language` parameter: `x86:LE:64:default` for 64-bit, `x86:LE:32:default` for 32-bit. |
| `ghidra_load_sample` returns "File not found" | Wrong path — container uses /data/ not /home/remnux/samples/ | The updated `ghidra-post.sh` translates paths automatically. If using `run_command` directly, use `/data/filename` not `/home/remnux/samples/filename`. |
| MCP not visible in Claude session | Registered with project scope | Re-register with `--scope user`. Check `~/.claude.json` for duplicates. |
| npm install hangs on REMnux/SIFT | No internet (host-only network) | Re-enable NAT temporarily to install. Switch back to host-only after. |
| sift-ssh not connecting | Wrong username or key path | SIFT user is `sansforensics`. Test: `ssh -i ~/.ssh/sift_key sansforensics@192.168.56.30`. |
| Volatility3 returns no output | Wrong OS profile or image path | Try `vol -f image.dmp windows.info` first. Use full absolute path. |
| netplan apply fails on SIFT | nano not installed | Use `sudo tee /etc/netplan/...` instead. Check interface name with `ip a`. |
| cfr/procyon empty output | Not a valid JAR or encrypted JAR | Verify with `file sample.jar`. Try `jadx` as fallback. |
| `go version -m` returns nothing | Stripped binary or not Go | Use `strings binary \| grep 'go[0-9]'`. Go binaries still contain module paths in strings even when stripped. |
| jadx not found | Not in PATH on REMnux | Use full path: `/usr/local/jadx/bin/jadx`. |

---

## 17. Security Notes

Rules to keep the lab isolated and your host machine safe.

- [ ] **Never execute samples on VM1.** VM1 holds your API keys and MCP config. Samples only ever live on REMnux (`/home/remnux/samples/`) or SIFT (`/cases/`).
- [ ] **Keep REMnux and SIFT host-only.** Switch back immediately after installing tools. Never leave analysis VMs on NAT during a session.
- [ ] **Use password-protected zips.** Always transfer samples as `zip -P infected sample.zip sample.exe`. This is the VT standard and prevents accidental execution.
- [ ] **Revert snapshots between samples.** Take a clean snapshot of REMnux before analysis. Revert after each session to prevent cross-contamination.
- [ ] **Disable shared clipboard and drag-and-drop** in VMware for REMnux and SIFT. Prevents accidental file transfer to the host.
- [ ] **API keys stay on VM1 only.** The VT API key is in `~/.config/malware-lab/secrets.env` on VM1. Never transfer to analysis VMs.
- [ ] **Hash all evidence.** Use `sift-ssh hash_file` before and after analysis to maintain chain of custody.
- [ ] **The Ghidra auth token is lab-internal only.** `ghidra-lab-token` binds to `127.0.0.1` inside Docker — not exposed on the lab network.

---

*AI-Powered Malware Analysis Lab — Installation Guide v1.1*
*VM1: Ubuntu 24.04 · VM2: REMnux · VM3: SIFT