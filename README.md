# AI-Powered Malware and Forensics Analysis Lab — Installation Guide

A three-VM isolated sandbox that combines Claude Code with REMnux analysis tools, Ghidra decompilation, and VirusTotal intelligence — all orchestrated through MCP servers from a clean Ubuntu host.

| | |
|---|---|
| **Hypervisor** | VMware Workstation / Fusion |
| **VMs** | Ubuntu 24.04 + REMnux + SIFT (Forensics) + Windows (future) |
| **LLM** | Claude Code (Sonnet) |

---

## Table of Contents

1. [Lab Architecture](#01-lab-architecture)
2. [Prerequisites](#02-prerequisites)
3. [VM1 — Ubuntu 24.04 (LLM Host)](#03-vm1--ubuntu-2404-llm-host)
4. [VM2 — REMnux (Analysis Machine)](#04-vm2--remnux-analysis-machine)
5. [Network Configuration](#05-network-configuration)
6. [SSH Key Setup](#06-ssh-key-setup)
7. [remnux-ssh MCP Server](#07-remnux-ssh-mcp-server)
8. [VirusTotal MCP Server](#08-virustotal-mcp-server)
9. [Ghidra MCP (Docker Headless)](#09-ghidra-mcp-docker-headless)
10. [REMnux Analysis Tools](#10-remnux-analysis-tools)
11. [MCP Tool Groups](#11-mcp-tool-groups)
12. [Analysis Workflows](#12-analysis-workflows)
13. [Troubleshooting](#13-troubleshooting)
14. [Security Notes](#14-security-notes)

---

## 01. Lab Architecture

Three VMs with strict network isolation. Only VM1 has internet access. Samples never touch the LLM host.

```
                    🌐 Internet / Anthropic API
                            │
   ┌─────────────────┐   SSH / Host-only    ┌─────────────────┐   SSH (future)   ┌─────────────────┐
   │ VM1              │   192.168.56.0/24    │ VM2              │  Windows         │ VM3              │
   │ Ubuntu 24.04     │ ───────────────────► │ REMnux           │  forensics       │ Windows 10       │
   │ Claude Code +    │                      │ Analysis Tools + │ ───────────────► │ Forensic         │
   │ MCP Servers      │                      │ Ghidra           │                  │ Analysis         │
   │ NAT + Host-only  │                      │ Host-only only   │                  │ Host-only (future)│
   └─────────────────┘                      └─────────────────┘                  └─────────────────┘
```

> **ℹ️ Key principle:** VM1 (Ubuntu) is the only machine with internet access. Malware samples are transferred to REMnux via SCP and never executed on VM1. The LLM orchestrates analysis remotely over SSH.

| VM | OS | Role | Network | Internet |
|----|----|------|---------|----------|
| `VM1` | Ubuntu 24.04 LTS | Claude Code + MCP orchestration host | NAT + Host-only `192.168.56.10` | ✅ Yes |
| `VM2` | REMnux (Ubuntu-based) | Static analysis, Ghidra, deobfuscation | Host-only only `192.168.56.20` | ❌ No |
| `VM3` | Windows 10 | Forensic analysis (future) | Host-only only `192.168.56.30` | ❌ No |

---

## 02. Prerequisites

Download everything before configuring the network. Both VMs need internet to install packages.

> **⚠️ Important:** Complete all downloads and package installations while VMs are on NAT (internet access). Switch REMnux to host-only *only* after all tools are installed.

### Host Machine Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 16 GB | 32 GB |
| Storage | 100 GB free | 200 GB SSD |
| CPU | 4 cores | 8 cores with VT-x |
| Hypervisor | VMware Workstation Pro / Fusion | VMware Workstation Pro / Fusion |

### Downloads Required

- [ ] **Ubuntu 24.04 LTS ISO** — [ubuntu.com/download](https://ubuntu.com/download/desktop) — for VM1
- [ ] **REMnux OVA** — [remnux.org](https://remnux.org) — pre-built VM with all analysis tools
- [ ] **VMware Workstation / Fusion** — [vmware.com](https://www.vmware.com)
- [ ] **Anthropic account** with Claude Pro, Max, Team, or Enterprise — needed for Claude Code authentication
- [ ] **VirusTotal API key** (Premium recommended for file downloads) — [virustotal.com](https://www.virustotal.com)

### VM Allocation

| VM | RAM | vCPU | Disk |
|----|-----|------|------|
| VM1 — Ubuntu | 4 GB | 2 | 40 GB |
| VM2 — REMnux | 4 GB | 2 | 60 GB |
| VM3 — SIFT | 4 GB | 2 | 60 GB |
| VM4 — Windows (future) | 4 GB | 2 | 60 GB |

---

## 03. VM1 — Ubuntu 24.04 (LLM Host)

Install Ubuntu 24.04, then install Claude Code and all MCP server dependencies. Keep VM1 on NAT throughout this section.

> **ℹ️** VM1 network adapter: set to **NAT** for now. You will add a second host-only adapter after installation.

### Step 1 — System Update

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git build-essential python3 python3-pip zip unzip
```

### Step 2 — Install Claude Code

Use the official native installer — no Node.js dependency required.

```bash
# Install Claude Code via native installer
curl -fsSL https://claude.ai/install.sh | bash

# Verify installation
claude --version
claude doctor
```

### Step 3 — Authenticate Claude Code

Authenticate using your corporate or personal Claude account via device code flow.

```bash
# Launch Claude — follow the device code prompt
claude

# When prompted:
# 1. Select "Sign in with Claude.ai"
# 2. Open the displayed URL in your browser
# 3. Enter the device code
# 4. Sign in with your corporate/personal account
```

### Step 4 — Install Node.js (for MCP servers)

```bash
# Install Node.js 20 LTS
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Verify
node --version   # v20.x.x
npm --version    # 10.x.x
```

### Step 5 — Store API Keys Securely

```bash
# Create secure secrets file
mkdir -p ~/.config/malware-lab
cat > ~/.config/malware-lab/secrets.env << 'EOF'
VIRUSTOTAL_API_KEY=paste_your_vt_key_here
EOF

# Lock down permissions
chmod 600 ~/.config/malware-lab/secrets.env

# Auto-load on login
echo 'source ~/.config/malware-lab/secrets.env' >> ~/.bashrc
source ~/.bashrc
```

### Step 6 — Create Working Directory

```bash
mkdir -p ~/malware-analysis ~/downloads

# Create CLAUDE.md to give Claude context about the lab
cat > ~/malware-analysis/CLAUDE.md << 'EOF'
# Malware Analysis Lab

## Startup
At the start of every session, list all available tools from remnux-ssh MCP grouped
by their category, then list all available virustotal MCP tools with a short description.
After showing the full list, wait for the next instruction.

## REMnux VM
- Host: 192.168.56.20
- User: remnux
- SSH key: /home/tasox/.ssh/remnux_key
- SSH command: ssh -i /home/tasox/.ssh/remnux_key remnux@192.168.56.20

## Directories
- Intake (zip files):  /home/remnux/intake/
- Samples (unzipped): /home/remnux/samples/
- Ghidra container maps /home/remnux/samples/ → /data/

## MCP Servers
- remnux-ssh  → all analysis tools on REMnux (triage, static, deobfuscation, Ghidra, .NET, ELF, Python, Office, MSI)
- virustotal  → hash lookups, detections, file reports, sample downloads (password: infected)
- ghidra-mcp → NOT used directly; use remnux-ssh ghidra_* tools instead

## Sample Transfer
To copy a sample to REMnux:
scp -i /home/tasox/.ssh/remnux_key <local_file> remnux@192.168.56.20:/home/remnux/intake/

## Analysis Workflow
1. Download sample via virustotal MCP → ~/downloads/sample.zip
2. Transfer: scp -i /home/tasox/.ssh/remnux_key ~/downloads/sample.zip remnux@192.168.56.20:/home/remnux/intake/
3. Unzip: remnux-ssh intake_sample with zip_path and password (default: infected)
4. Triage: use the appropriate triage_* tool based on file type
5. Deep analysis: ghidra_load_sample → ghidra_* tools
6. Intelligence: virustotal get_file_report with the sample hash

## Rules
- Never store or execute samples on VM1 (this machine)
- Samples only live in /home/remnux/samples/ on REMnux
- Always revert REMnux snapshot after each analysis session
EOF
```

### Step 7 — Create the `lab` Shortcut

This shell function starts a Claude session from the analysis directory and automatically displays all available tools at startup.

```bash
# Add lab function to ~/.bashrc
cat >> ~/.bashrc << 'EOF'

lab() {
  cd ~/malware-analysis
  echo "Loading malware analysis lab..."
  claude "Show all available tools for this malware analysis lab grouped by category:
1. List all tools from remnux-ssh MCP grouped by their [CATEGORY] tag
2. List all tools from virustotal MCP with a short description of each
After showing the full list, wait for my next instruction."
}
EOF

source ~/.bashrc

# Start the lab — shows all tools then waits for your prompt
lab
```

> **✅** From now on, type `lab` from anywhere to start an analysis session. Claude will automatically list all available tools grouped by category before waiting for your first prompt.

---

## 04. VM2 — REMnux (Analysis Machine)

Import the REMnux OVA into VMware, then install additional tools while NAT is still active. Switch to host-only after all tools are installed.

> **⚠️** Keep REMnux on **NAT** during this entire section. You will switch to host-only in the Network Configuration section.

### Step 1 — Import REMnux OVA

In VMware: **File → Open** → select the REMnux `.ova` file → Import. Allocate 4 GB RAM and 2 vCPUs.

### Step 2 — Install SSH Server

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh

# Verify SSH is listening
sudo ss -tlnp | grep 22
```

### Step 3 — Install Additional Analysis Tools

```bash
# Python analysis tools
pip3 install vipermonkey uncompyle6 --break-system-packages

# JavaScript deobfuscation
npm install -g synchrony js-beautify

# Verify key tools are available
which olevba oleid mraptor diec floss capa
which readelf objdump nm binwalk exiftool
which js-beautify synchrony
```

### Step 4 — Install Docker (for Ghidra MCP)

```bash
# Docker is typically pre-installed on REMnux — verify:
docker --version

# If not installed:
sudo apt install -y docker.io
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
```

### Step 5 — Create Sample Directories

```bash
mkdir -p /home/remnux/samples
mkdir -p /home/remnux/intake
mkdir -p /home/remnux/ghidra-scripts
```

---

## 05. Network Configuration

Configure VMware network adapters and static IPs. Only do this after all tools are installed on both VMs.

### VMware Adapter Configuration

| VM | Adapter 1 | Adapter 2 |
|----|-----------|-----------|
| VM1 — Ubuntu | NAT — internet access | Host-only VMnet1 — lab network |
| VM2 — REMnux | Host-only VMnet1 — lab network only | — |

In VMware: **VM → Settings → Network Adapter**. For VM1, add a second adapter via **Add → Network Adapter → Host-only**.

### Assign Static IPs on VM1 (Ubuntu)

```bash
# Find the host-only interface name
ip a
# Look for the second adapter (not your NAT one) — typically ens36 or similar

# Edit netplan config
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

### Assign Static IP on VM2 (REMnux)

`/etc/netplan/01-network-manager-all.yaml` (VM2):

```yaml
network:
  version: 2
  ethernets:
    ens33:           # Host-only adapter — static IP
      dhcp4: no
      addresses: [192.168.56.20/24]
```

```bash
sudo netplan apply

# Verify connectivity from VM1
ping -c 3 192.168.56.10   # ping VM1 from VM2
```

> **✅** Both VMs should be able to ping each other on `192.168.56.0/24`. VM1 should also still have internet access via its NAT adapter.

---

## 06. SSH Key Setup

Generate SSH keys on VM1 and copy them to REMnux for passwordless access. This is required for the MCP servers to communicate.

**1. Generate SSH key on VM1**

```bash
ssh-keygen -t ed25519 -f ~/.ssh/remnux_key -N "" -C "vm1-to-remnux"
```

**2. Copy public key to REMnux**

```bash
ssh-copy-id -i ~/.ssh/remnux_key.pub remnux@192.168.56.20
```

**3. Create SSH config for convenience** (`~/.ssh/config` on VM1)

```
Host remnux
    HostName 192.168.56.20
    User remnux
    IdentityFile ~/.ssh/remnux_key
    StrictHostKeyChecking no
```

**4. Test connectivity**

```bash
ssh remnux "uname -a && echo SSH OK"
```

---

## 07. remnux-ssh MCP Server

Build and register the custom MCP server that lets Claude Code run commands on REMnux over SSH.

### Build the MCP Server

```bash
# Create MCP directory
mkdir -p ~/mcp/remnux-mcp/src

# Install TypeScript dependencies
cd ~/mcp/remnux-mcp
npm init -y
npm install @modelcontextprotocol/sdk ssh2 zod
npm install -D typescript @types/node @types/ssh2

# Configure TypeScript
cat > tsconfig.json << 'EOF'
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "build",
    "rootDir": "src",
    "strict": false,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}
EOF

# Add "type": "module" to package.json
npm pkg set type="module"
```

Place the `index.ts` source file into `~/mcp/remnux-mcp/src/index.ts`, then build:

```bash
cd ~/mcp/remnux-mcp
npx tsc

# Verify build output
ls build/
```

### Create Ghidra Helper Scripts on REMnux

These scripts solve SSH quoting issues when calling the Ghidra Docker container.

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
echo \x27{\"file\":\"\x27\$1\x27\"}\x27 > /tmp/gh_body.json
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

### Register remnux-ssh with Claude Code

```bash
claude mcp add remnux-ssh \
  --scope user \
  -- node /home/$USER/mcp/remnux-mcp/build/index.js \
  --host 192.168.56.20 \
  --username remnux \
  --private-key /home/$USER/.ssh/remnux_key

# Verify
claude mcp list | grep remnux-ssh
```

---

## 08. VirusTotal MCP Server

Set up the VirusTotal MCP for hash lookups, file reports, and sample downloads.

> **ℹ️** File download requires a **VirusTotal Premium API key**. Hash lookups and reports work with a free key.

```bash
# Clone and build the VT MCP
git clone https://github.com/BurtTheCoder/mcp-virustotal ~/mcp/mcp-virustotal
cd ~/mcp/mcp-virustotal
npm install
npm run build

# Register with Claude Code (loads API key from secrets file)
source ~/.config/malware-lab/secrets.env
claude mcp add virustotal \
  --scope user \
  -e VIRUSTOTAL_API_KEY=$VIRUSTOTAL_API_KEY \
  -- node /home/$USER/mcp/mcp-virustotal/build/index.js

# Test — EICAR hash (safe to query)
# Open a claude session and run:
# "Use virustotal MCP to look up hash: 44d88612fea8a8f36de82e1278abb02f"
```

---

## 09. Ghidra MCP (Docker Headless)

Run GhidraMCP as a headless Docker container on REMnux for automated binary analysis. The container is built on VM1 (which has internet) and transferred to REMnux.

### Step 1 — Build Docker Image on VM1

```bash
# Install Docker on VM1
sudo apt install -y docker.io
sudo systemctl start docker
sudo usermod -aG docker $USER && newgrp docker

# Clone ghidra-mcp repo
git clone https://github.com/bethington/ghidra-mcp ~/mcp/ghidra-mcp
cd ~/mcp/ghidra-mcp

# Build the Docker image (takes 5-10 minutes)
docker build -t ghidra-mcp-headless:latest -f docker/Dockerfile .

# Export image to tar
docker save ghidra-mcp-headless:latest -o /tmp/ghidra-mcp-headless.tar
ls -lh /tmp/ghidra-mcp-headless.tar
```

### Step 2 — Transfer Image to REMnux

```bash
scp -i ~/.ssh/remnux_key /tmp/ghidra-mcp-headless.tar remnux@192.168.56.20:/home/remnux/
```

### Step 3 — Load and Run Container on REMnux

```bash
# Load image
docker load -i /home/remnux/ghidra-mcp-headless.tar

# Create run script
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
with open('/home/remnux/run-ghidra-mcp.sh', 'w') as f:
    f.write(script)
os.chmod('/home/remnux/run-ghidra-mcp.sh', 0o755)
print('Script written')
"

# Start the container
/home/remnux/run-ghidra-mcp.sh

# Verify it's healthy (~20 seconds startup)
sleep 20
docker exec ghidra-mcp curl -s http://localhost:8089/get_version
```

> **✅** Expected output: `{"plugin_version": "5.13.1-headless", "plugin_name": "GhidraMCP Headless", "mode": "headless"}`

---

## 10. REMnux Analysis Tools

Complete list of analysis tools available on REMnux, organized by category.

### Static Analysis

| Tool | Description | Source |
|------|-------------|--------|
| `file` | Identify file type by magic bytes | built-in |
| `strings` | Extract printable strings | built-in |
| `xxd` | Hex dump files | built-in |
| `diec` / `die` | Detect packers, compilers, protectors | pre-installed |
| `FLOSS` | Extract obfuscated + stack strings | pre-installed |
| `CAPA` | MITRE ATT&CK capability detection | pre-installed |
| `pescanner` | PE header analysis | pre-installed |
| `exiftool` | File metadata extraction | pre-installed |
| `binwalk` | Embedded file extraction | pre-installed |
| `upx` | UPX packer detection / unpacking | pre-installed |

### Office Document Analysis

| Tool | Description | Source |
|------|-------------|--------|
| `olevba` | VBA macro extraction and analysis | oletools |
| `oleid` | OLE file indicator detection | oletools |
| `oledump` | OLE stream dumping | oletools |
| `mraptor` | Malicious macro detection | oletools |
| `ViperMonkey` | VBA macro emulation | install |

### Script Deobfuscation

| Tool | Description | Source |
|------|-------------|--------|
| `synchrony` | JavaScript deobfuscation (obfuscator.io) | install |
| `js-beautify` | JavaScript formatting | install |

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

### Windows Installer

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
| `dnfile` | Python library for .NET metadata parsing — assembly refs, type defs, method names | install |
| `pefile` | Python PE structure parsing library | install |
| `dotnet runtime` | Microsoft .NET 8.0 runtime (required by ilspycmd) | pre-installed |

### Install .NET Python Libraries

```bash
pip3 install dnfile pefile --break-system-packages

# Verify
python3 -c "import dnfile, pefile; print('dnfile', dnfile.__version__, 'pefile ok')"
```

---

## 11. MCP Tool Groups

The remnux-ssh MCP exposes tools organized into groups. Each group tag appears in the tool description so Claude picks the right tool automatically.

| Group | Tag | Tools | Use when |
|-------|-----|-------|----------|
| Intake & Utility | `[UTILITY]` | `run_command`, `upload_sample`, `intake_sample`, `list_samples` | Transferring and managing samples |
| Quick Triage | `[TRIAGE]` | `triage_quick`, `triage_pe`, `triage_iocs`, `triage_office`, `triage_script`, `triage_elf`, `triage_python`, `triage_full` | First look at any new sample |
| Deep Static | `[STATIC]` | `analyze_file`, `floss_strings`, `capa_analyze`, `diec_analyze`, `binwalk_extract`, `extract_config` | Detailed analysis after triage |
| Office Documents | `[OFFICE]` | `oleid_analyze`, `olevba_analyze`, `oledump_analyze`, `mraptor_analyze`, `vmonkey_analyze` | doc/docx/xls/xlsx/ppt samples |
| Script Deobfuscation | `[DEOBFUSCATE]` | `deobfuscate_js`, `js_beautify`, `js_extract_strings`, `deobfuscate_powershell` | JS/PS1/VBS scripts |
| MSI / MSIX | `[MSI]` | `msi_analyze`, `msi_extract_files`, `msix_analyze` | Windows installer packages |
| ELF Analysis | `[ELF]` | `elf_analyze`, `elf_imports`, `elf_sections`, `elf_iocs` | Linux ELF binaries |
| Python Malware | `[PYTHON]` | `pyinstaller_extract`, `python_decompile`, `python_analyze` | Compiled Python samples |
| Ghidra | `[GHIDRA]` | `ghidra_version`, `ghidra_load_sample`, `ghidra_list_functions`, `ghidra_decompile`, `ghidra_list_imports`, `ghidra_list_strings`, `ghidra_detect_malware`, `ghidra_extract_iocs` | Deep reverse engineering |
| .NET Assembly | `[DOTNET]` | `dotnet_decompile`, `dotnet_il_disasm`, `dotnet_pe_headers`, `dotnet_metadata`, `dotnet_strings`, `dotnet_resources`, `triage_dotnet` | C#, VB.NET, F# compiled assemblies |

---

## 12. Analysis Workflows

Standard prompts to use with Claude Code for common analysis scenarios. Run these from the `~/malware-analysis` directory.

### Workflow 1 — Full PE Sample Analysis

```
Full malware analysis workflow:

1. Use virustotal MCP to download SHA256 <HASH> to ~/downloads/sample.zip
2. Transfer sample.zip to REMnux: scp to /home/remnux/intake/sample.zip
3. Use remnux-ssh intake_sample to unzip with password infected
4. Use remnux-ssh triage_full on the extracted sample
5. Use virustotal get_file_report for the hash
6. Use remnux-ssh ghidra_load_sample then ghidra_list_imports and ghidra_detect_malware
7. Write a triage report summarizing: file type, hashes, capabilities,
   IOCs, VT detections, and recommended next steps
```

### Workflow 2 — Office Document Analysis

```
Analyze the Office document at /home/remnux/samples/invoice.docx:

1. Use remnux-ssh triage_office to run all oletools
2. Use remnux-ssh vmonkey_analyze to emulate any macros found
3. Extract all IOCs (URLs, IPs, domains) from the document
4. Summarize: macro presence, auto-exec triggers, network IOCs, verdict
```

### Workflow 3 — JavaScript Deobfuscation

```
Deobfuscate the JavaScript at /home/remnux/samples/malicious.js:

1. Use remnux-ssh triage_script on the file
2. Use remnux-ssh deobfuscate_js with method=both
3. Extract all network IOCs from the deobfuscated output
4. Identify the malware family if possible
```

### Workflow 4 — PyInstaller Sample

```
Analyze the Python binary at /home/remnux/samples/stealer.exe:

1. Use remnux-ssh triage_python on the file
2. If PyInstaller detected, use pyinstaller_extract to unpack it
3. Decompile any .pyc files found with python_decompile
4. Extract C2 configuration from the decompiled source
5. Cross-reference IOCs with virustotal
```

### Workflow 5 — .NET Assembly Analysis

```
Analyze the .NET sample at /home/remnux/samples/agent.exe:

1. Use remnux-ssh triage_dotnet for a full initial analysis
2. Use remnux-ssh dotnet_decompile to get the full C# source
3. Use remnux-ssh dotnet_strings to find obfuscated strings and C2 indicators
4. Use remnux-ssh dotnet_resources to check for embedded payloads
5. Use virustotal get_file_report for detections and community intel
6. Summarize: obfuscator used, capabilities, IOCs, verdict
```

### VirusTotal MCP — Available Tools

These tools are available via the `virustotal` MCP in any Claude session started with `lab`.

| Tool | Input | Description | API tier |
|------|-------|-------------|----------|
| `get_file_report` | MD5 / SHA1 / SHA256 | Full detection report — AV verdicts, sandbox scores, community ratings, file metadata | Free |
| `get_file_behavior` | SHA256 | Dynamic analysis summary — network connections, registry changes, dropped files | Free |
| `get_file_relationship` | SHA256 + relationship type | Related files, dropped files, contacted IPs/domains, execution parents | Free |
| `get_url_report` | URL | URL reputation, category, detections, last analysis results | Free |
| `get_ip_report` | IP address | IP reputation, ASN, country, associated files and URLs | Free |
| `get_domain_report` | Domain | Domain reputation, WHOIS, DNS history, associated malware | Free |
| `search` | VTI query string | Advanced search — supports VTI modifiers e.g. `type:peexe tag:signed` | Free |
| `download_file` | SHA256 + output path | Download sample as password-protected zip (password: `infected`) to VM1 | Premium |

> **ℹ️** The `download_file` tool saves the sample to `~/downloads/` on VM1 as a zip. Always transfer it to REMnux via `scp` before unzipping — never extract samples directly on VM1.

---

## 13. Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| SSH connection refused | SSH service not running or wrong IP | Run `sudo systemctl start ssh` on REMnux. Verify IP with `ip a`. |
| MCP shows "Failed to connect" | Node.js path wrong or build not compiled | Run `npx tsc` in `~/mcp/remnux-mcp`. Re-register with `claude mcp remove` then `add`. |
| Ghidra returns "Unauthorized" | Auth token not passed correctly through SSH layers | Use the ghidra-scripts wrapper approach — do not pass token directly via `docker exec`. |
| Ghidra container crashes on start | Binding to 0.0.0.0 without auth token | Always set `GHIDRA_MCP_AUTH_TOKEN` when using `GHIDRA_MCP_BIND_ADDRESS=0.0.0.0`. |
| MCP not visible in Claude session | Registered with project scope, not user scope | Re-register with `--scope user` flag. Check `~/.claude.json` for duplicate entries. |
| SSH tunnel lost after reboot | Tunnel not persistent | Add tunnel command to `~/.bashrc`: `ssh -i ~/.ssh/remnux_key -N -L 8089:127.0.0.1:8089 remnux@192.168.56.20 &` |
| npm install hangs on REMnux | Registry blocked (REMnux has no internet) | Install on VM1, then `scp node_modules` to REMnux. Or temporarily enable NAT. |

---

## 14. Security Notes

Rules to keep the lab isolated and your host machine safe.

- [ ] **Never execute samples on VM1.** VM1 is the LLM host — it has your API keys and MCP config. Samples only ever live on REMnux (`/home/remnux/samples/`).
- [ ] **Keep REMnux host-only.** Switch REMnux back to host-only immediately after installing tools. A single NAT session is fine for installs, but leave it off during analysis.
- [ ] **Use password-protected zips.** Always transfer samples as `zip -P infected sample.zip sample.exe`. This prevents accidental execution and is the VT standard.
- [ ] **Revert snapshots between samples.** Take a clean snapshot of REMnux before analysis. Revert after each session to ensure no persistence from previous samples.
- [ ] **Disable shared clipboard and drag-and-drop** in VMware settings for REMnux. This prevents accidental file transfer from the analysis VM to your host.
- [ ] **API keys stay on VM1 only.** The VirusTotal API key is stored in `~/.config/malware-lab/secrets.env` on VM1. It is never transferred to REMnux.
- [ ] **The Ghidra auth token is lab-internal only.** `ghidra-lab-token` is bound to `127.0.0.1` inside the Docker container — it is not exposed on the network.

---

*AI-Powered Malware Analysis Lab — Installation Guide v1.0*
*VM1: Ubuntu 24.04 · VM2: REMnux · LLM: Claude Code*