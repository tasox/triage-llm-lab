# AI-Powered Malware Analysis Lab — Installation Guide

A four-VM isolated sandbox that combines Claude Code with REMnux analysis tools, SIFT forensic workstation, GhidraMCP headless decompilation, VirusTotal intelligence, and Windows dynamic analysis — all orchestrated through MCP servers from a clean Ubuntu host.

| | |
|---|---|
| **Hypervisor** | VMware Workstation / Fusion |
| **VMs** | Ubuntu + REMnux + SIFT + Windows 11 |
| **LLM** | Claude Code |

---

## Table of Contents

1. [Lab Architecture](#01-lab-architecture)
2. [Prerequisites](#02-prerequisites)
3. [VM1 — Ubuntu 24.04 (LLM Host)](#03-vm1--ubuntu-2404-llm-host)
4. [VM2 — REMnux (Analysis Machine)](#04-vm2--remnux-analysis-machine)
5. [VM3 — SIFT Workstation (Forensics)](#05-vm3--sift-workstation-forensics)
6. [VM4 — Windows 11 (Dynamic Analysis)](#05b-vm4--windows-11-dynamic-analysis)
7. [Network Configuration](#06-network-configuration)
8. [SSH Key Setup](#07-ssh-key-setup)
9. [remnux-ssh MCP Server](#08-remnux-ssh-mcp-server)
10. [VirusTotal MCP Server](#09-virustotal-mcp-server)
11. [Ghidra MCP (Docker Headless)](#10-ghidra-mcp-docker-headless)
12. [sift-ssh MCP Server](#11-sift-ssh-mcp-server)
13. [x64dbg MCP Server](#11b-x64dbg-mcp-server)
14. [WinDbg MCP Server](#11c-windbg-mcp-server)
15. [REMnux Analysis Tools](#12-remnux-analysis-tools)
16. [SIFT Forensic Tools](#13-sift-forensic-tools)
17. [Windows Dynamic Analysis Tools](#13b-windows-dynamic-analysis-tools)
18. [MCP Tool Groups](#14-mcp-tool-groups)
19. [Analysis Workflows](#15-analysis-workflows)
20. [Troubleshooting](#16-troubleshooting)
21. [Security Notes](#17-security-notes)

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
   │                   │                      │ Windows 11             │
   │                   │                      │ Dynamic Analysis +     │
   │                   │                      │ x64dbg + WinDbg        │
   └──────────────────┘                      │ Host-only · .56.40     │
                                              └──────────────────────┘
```

> **Key principle:** VM1 (Ubuntu) is the only machine with internet access. Samples are transferred to REMnux or SIFT via SCP and never executed on VM1. Claude orchestrates all analysis remotely over SSH.

| VM | OS | Role | IP | Internet |
|----|----|------|----|----------|
| `VM1` | Ubuntu 24.04 LTS | Claude Code + MCP orchestration host | `192.168.56.10` | ✅ Yes |
| `VM2` | REMnux (Ubuntu-based) | Static analysis, Ghidra, deobfuscation, malware triage | `192.168.56.20` | ❌ No |
| `VM3` | SIFT Workstation (Ubuntu 22.04) | Disk forensics, memory analysis, timeline, registry | `192.168.56.30` | ❌ No |
| `VM4` | Windows 11 | Dynamic analysis, x64dbg debugging, WinDbg, FakeNet-NG | `192.168.56.40` | ❌ No |

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

- [ ] **Ubuntu 24.04 LTS ISO** — for VM1
- [ ] **REMnux OVA** — pre-built analysis VM
- [ ] **SIFT Workstation OVA** — requires free SANS account. Default login: `sansforensics` / `forensics`
- [ ] **Windows 11 ISO** — for VM4 dynamic analysis
- [ ] **x64dbg** — Windows debugger with MCP plugin
- [ ] **x64dbg MCP plugin** — `x64dbg-mcp` from [github.com/SetsunaYukiOvO/x64dbg-mcp](https://github.com/SetsunaYukiOvO/x64dbg-mcp)
- [ ] **FakeNet-NG 3.5** — pre-built executable from [github.com/mandiant/flare-fakenet-ng/releases/tag/v3.5](https://github.com/mandiant/flare-fakenet-ng/releases/tag/v3.5)
- [ ] **mcp-windbg** — `pip install mcp-windbg` on VM4
- [ ] **VMware Workstation / Fusion**
- [ ] **Anthropic account** with Claude Pro, Max, Team, or Enterprise subscription
- [ ] **VirusTotal API key** — Premium recommended for file downloads

### VM Allocation

| VM | RAM | vCPU | Disk | Status |
|----|-----|------|------|--------|
| VM1 — Ubuntu 24.04 | 4 GB | 2 | 40 GB | ✅ Active |
| VM2 — REMnux | 4 GB | 2 | 60 GB | ✅ Active |
| VM3 — SIFT Workstation | 4 GB | 2 | 80 GB | ✅ Active |
| VM4 — Windows 11 | 4 GB | 2 | 80 GB | ✅ Active |

---

## 03. VM1 — Ubuntu 24.04 (LLM Host)

Install Ubuntu 24.04, then install Claude Code and all MCP server dependencies. Keep VM1 on NAT throughout this section.

### Step 1 — System Update

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git build-essential python3 python3-pip zip unzip
```

### Step 2 — Install Claude Code

```bash
curl -fsSL https://claude.ai/install.sh | bash
claude --version
claude doctor
```

### Step 3 — Authenticate Claude Code

```bash
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
4. x64dbg MCP tools grouped by their category
5. windbg MCP tools grouped by their category
Then wait for the next instruction.

## VMs
- VM1 Ubuntu:  192.168.56.10 — this machine (Claude Code + MCP host)
- VM2 REMnux:  192.168.56.20 — static analysis, Ghidra
- VM3 SIFT:    192.168.56.30 — disk/memory/timeline forensics
- VM4 Windows: 192.168.56.40 — dynamic analysis, x64dbg, WinDbg

## SSH Keys & Commands
- ~/.ssh/remnux_key  → VM2: ssh -i ~/.ssh/remnux_key remnux@192.168.56.20
- ~/.ssh/sift_key    → VM3: ssh -i ~/.ssh/sift_key sansforensics@192.168.56.30
- ~/.ssh/windows_key → VM4: ssh -i ~/.ssh/windows_key analyst@192.168.56.40

## MCP Servers
- remnux-ssh  → static analysis tools on REMnux
- sift-ssh    → forensic tools on SIFT
- virustotal  → hash lookups, detections, sample downloads (password: infected)
- ghidra-mcp  → NOT used directly; use remnux-ssh ghidra_* tools instead
- x64dbg      → debugger control on Windows VM (via SSH tunnel port 3000)
- windbg      → CDB/WinDbg crash dump analysis on Windows VM (via SSH tunnel port 8000)

## Sample Transfer
scp -i ~/.ssh/remnux_key <file> remnux@192.168.56.20:/home/remnux/intake/
scp -i ~/.ssh/sift_key <file> sansforensics@192.168.56.30:/cases/
scp -i ~/.ssh/windows_key <file> analyst@192.168.56.40:C:/samples/

## Directories — REMnux (VM2)
- Intake:   /home/remnux/intake/
- Samples:  /home/remnux/samples/
- Ghidra container maps /home/remnux/samples/ → /data/

## Directories — SIFT (VM3)
- Evidence: /cases/   Memory: /cases/*.dmp   Disk: /cases/*.dd

## Directories — Windows (VM4)
- Samples:  C:\samples\
- Dumps:    C:\dumps\

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

## Dynamic Analysis Workflow (Windows)
1. Start FakeNet on VM4 manually before detonating
2. Transfer: scp -i ~/.ssh/windows_key <sample> analyst@192.168.56.40:C:/samples/
3. x64dbg: debug_init → breakpoint_set on APIs → debug_run
4. Unpack: dump_detect_oep → dump_module
5. Transfer dump back to REMnux for static analysis

## Ghidra Headless Server (via remnux-ssh)
- Auth: Authorization: Bearer ghidra-lab-token
- Load PE/ELF: ghidra_load_sample with path only (auto-detects)
- Load shellcode/raw: ghidra_load_sample with path + language
  - x86 64-bit:  x86:LE:64:default
  - x86 32-bit:  x86:LE:32:default
  - ARM 32-bit:  ARM:LE:32:v8
  - ARM 64-bit:  AARCH64:LE:64:v8A
- After raw blobs: call ghidra_run_analysis with entry_offset=0x0
- Path mapping: /home/remnux/samples/ → /data/ (handled by ghidra-post.sh)

## Rules
- Never store or execute samples on VM1
- Samples only live in /home/remnux/samples/ on REMnux
- Evidence only in /cases/ on SIFT
- Windows samples in C:\samples\ on VM4
- Always revert REMnux and Windows snapshots after analysis
EOF
```

### Step 7 — Create the `lab` Shortcut

```bash
cat >> ~/.bashrc << 'EOF'

lab() {
  cd ~/malware-analysis
  # Start SSH tunnels — x64dbg (3000) and WinDbg (8000)
  ssh -i ~/.ssh/windows_key -N \
    -L 3000:127.0.0.1:3000 \
    -L 8000:127.0.0.1:8000 \
    analyst@192.168.56.40 &>/dev/null &
  sleep 3  # wait for tunnels to establish
  echo "Loading malware analysis lab..."
  claude "Show all available tools for this malware analysis lab grouped by category:
1. List all tools from remnux-ssh MCP grouped by their [CATEGORY] tag
2. List all tools from sift-ssh MCP grouped by their [CATEGORY] tag
3. List all tools from virustotal MCP with a short description of each
4. List all tools from x64dbg MCP grouped by their category
5. List all tools from windbg MCP grouped by their category
After showing the full list, wait for my next instruction."
}
EOF
source ~/.bashrc
```

> Type `lab` from anywhere to start an analysis session. The SSH tunnels to VM4 start automatically.

---

## 04. VM2 — REMnux (Analysis Machine)

Import the REMnux OVA, install all tools while NAT is active, then switch to host-only.

> **⚠️** Keep REMnux on **NAT** during this entire section.

### Step 1 — Import REMnux OVA

In VMware: **File → Open** → select the REMnux `.ova` → Import. Allocate 4 GB RAM, 2 vCPUs.

### Step 2 — Install SSH Server

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable ssh && sudo systemctl start ssh
sudo ss -tlnp | grep 22
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

> Most tools are pre-installed on REMnux. Run the commands above — existing installs will simply be skipped.

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

> After applying the static IP, switch the VMware adapter to **Host-only (VMnet1)**.

---

## 05b. VM4 — Windows 11 (Dynamic Analysis)

Install Windows 11, configure OpenSSH, set static IP, install x64dbg with MCP plugin, mcp-windbg, and FakeNet-NG. Keep on NAT during installation.

> **⚠️** Keep Windows 11 on **NAT** during this section.

### Step 1 — Install OpenSSH Server

```powershell
# Run in PowerShell (Admin)
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server' `
  -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
New-LocalUser -Name "analyst" -Password (ConvertTo-SecureString "LabPassword123!" -AsPlainText -Force)
Add-LocalGroupMember -Group "Administrators" -Member "analyst"
```

### Step 2 — Set Static IP

```powershell
Get-NetAdapter  # find adapter name
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 192.168.56.40 -PrefixLength 24
```

Then in VMware: **VM → Settings → Network Adapter → Host-only (VMnet1)**

### Step 3 — Configure SSH Key Authentication

```powershell
# Run in PowerShell (Admin)
New-Item -ItemType Directory -Path "C:\Users\analyst\.ssh" -Force
icacls "C:\Users\analyst\.ssh" /inheritance:r
icacls "C:\Users\analyst\.ssh" /grant "analyst:(OI)(CI)F"

# Paste public key from VM1 (cat ~/.ssh/windows_key.pub)
$pubkey = "paste_content_of_windows_key.pub_here"
Set-Content -Path "C:\Users\analyst\.ssh\authorized_keys" -Value $pubkey
icacls "C:\Users\analyst\.ssh\authorized_keys" /inheritance:r
icacls "C:\Users\analyst\.ssh\authorized_keys" /grant "analyst:F"
icacls "C:\Users\analyst\.ssh\authorized_keys" /grant "SYSTEM:F"

# Disable administrator_authorized_keys override in sshd_config
$c = "C:\ProgramData\ssh\sshd_config"
(Get-Content $c) -replace '^Match Group administrators','#Match Group administrators' | Set-Content $c
Restart-Service sshd
```

### Step 4 — Install x64dbg + MCP Plugin

- Download x64dbg from [x64dbg.com](https://x64dbg.com) → extract to `C:\Users\tasox\Downloads\x64dbg\`
- Download MCP plugin from [github.com/SetsunaYukiOvO/x64dbg-mcp](https://github.com/SetsunaYukiOvO/x64dbg-mcp)
- Place `x64dbg_mcp.dp64` → `x64dbg\release\x64\plugins\`
- Place `x64dbg_mcp.dp32` → `x64dbg\release\x32\plugins\`
- Plugin starts HTTP server on `127.0.0.1:3000` automatically when x64dbg loads

### Step 5 — Install mcp-windbg

```powershell
pip install mcp-windbg
mcp-windbg --help
```

### Step 6 — Install FakeNet-NG

- Download `fakenet.exe` from [flare-fakenet-ng releases v3.5](https://github.com/mandiant/flare-fakenet-ng/releases/tag/v3.5)
- Extract to `C:\Users\tasox\Downloads\fakenet3.5\fakenet3.5\`
- **Do not autostart** — launch manually before each analysis session

### Step 7 — Create Autostart Script

```powershell
New-Item -ItemType Directory -Path "C:\lab" -Force

@'
Start-Process "C:\Users\tasox\Downloads\x64dbg\release\x64\x64dbg.exe" -WindowStyle Normal
Start-Process "C:\Users\tasox\Downloads\x64dbg\release\x32\x32dbg.exe" -WindowStyle Normal
Start-Sleep -Seconds 5
Start-Process "C:\Users\tasox\AppData\Local\Programs\Python\Python313\Scripts\mcp-windbg.exe" `
  -ArgumentList "--transport streamable-http --host 127.0.0.1 --port 8000"
'@ | Out-File "C:\lab\startup.ps1" -Encoding UTF8

$action = New-ScheduledTaskAction -Execute "powershell.exe" `
  -Argument "-ExecutionPolicy Bypass -WindowStyle Hidden -File C:\lab\startup.ps1"
$trigger = New-ScheduledTaskTrigger -AtLogOn -User "tasox"
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries
Register-ScheduledTask -TaskName "MalwareLab-Startup" `
  -Action $action -Trigger $trigger -Settings $settings -RunLevel Highest -Force
Start-ScheduledTask -TaskName "MalwareLab-Startup"
```

> x64dbg and mcp-windbg start automatically at login. FakeNet must be started manually before detonating samples.

---

## 06. Network Configuration

Configure VMware adapters and static IPs. Do this after all tools are installed.

### VMware Adapter Configuration

| VM | Adapter 1 | Adapter 2 |
|----|-----------|-----------|
| VM1 — Ubuntu | NAT — internet access | Host-only VMnet1 — lab network |
| VM2 — REMnux | Host-only VMnet1 — lab network only | — |
| VM3 — SIFT | Host-only VMnet1 — lab network only | — |
| VM4 — Windows | Host-only VMnet1 — lab network only | — |

### Static IP on VM1 (Ubuntu)

`/etc/netplan/01-network-manager-all.yaml`:

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

> VM1, VM2, VM3 and VM4 should all ping each other on `192.168.56.0/24`. VM1 should also have internet via NAT.

---

## 07. SSH Key Setup

Generate SSH keys on VM1 for all three analysis VMs.

**1. Generate keys on VM1**

```bash
ssh-keygen -t ed25519 -f ~/.ssh/remnux_key  -N "" -C "vm1-to-remnux"
ssh-keygen -t ed25519 -f ~/.ssh/sift_key    -N "" -C "vm1-to-sift"
ssh-keygen -t ed25519 -f ~/.ssh/windows_key -N "" -C "vm1-to-windows"
```

**2. Copy keys to VMs**

```bash
ssh-copy-id -i ~/.ssh/remnux_key.pub remnux@192.168.56.20
ssh-copy-id -i ~/.ssh/sift_key.pub   sansforensics@192.168.56.30
cat ~/.ssh/windows_key.pub  # paste this into VM4 authorized_keys (see VM4 Step 3)
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

Host windows
    HostName 192.168.56.40
    User analyst
    IdentityFile ~/.ssh/windows_key
    StrictHostKeyChecking no
```

**4. Test connectivity**

```bash
ssh remnux "uname -a && echo REMnux OK"
ssh sift "uname -a && vol -h 2>/dev/null | head -2 && echo SIFT OK"
ssh windows "whoami && hostname && echo Windows OK"
```

---

## 08. remnux-ssh MCP Server

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

> The POST script maps `/home/remnux/samples/` → `/data/` automatically. For raw shellcode, pass the language as a second argument e.g. `x86:LE:64:default`.

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

> File download requires a **VirusTotal Premium API key**. Hash lookups and reports work with a free key.

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
| `search` | VTI query | Advanced search with VTI modifiers e.g. `type:peexe tag:signed` | Free |
| `download_file` | SHA256 + path | Download sample as password-protected zip (password: `infected`) | Premium |

> `download_file` saves to `~/downloads/` on VM1. Always transfer to REMnux via `scp` before unzipping.

---

## 10. Ghidra MCP (Docker Headless)

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
# On REMnux
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

Expected: `{"plugin_version": "5.13.1-headless", "mode": "headless"}`

---

## 11. sift-ssh MCP Server

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

claude mcp list | grep sift
```

> Test with: *"Use sift-ssh run_command to run: uname -a && vol -h 2>/dev/null | head -2"*

---

## 11b. x64dbg MCP Server

x64dbg MCP runs on VM4 and is accessed from VM1 via an SSH tunnel. The tunnel must be active before starting a Claude session.

### Step 1 — Start SSH Tunnel from VM1

```bash
# Start tunnel for both x64dbg (3000) and WinDbg (8000)
ssh -i ~/.ssh/windows_key -N \
  -L 3000:127.0.0.1:3000 \
  -L 8000:127.0.0.1:8000 \
  analyst@192.168.56.40 &

sleep 3  # wait for tunnel to establish

# Test x64dbg MCP is responding
curl -s -X POST http://127.0.0.1:3000/rpc \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' | python3 -m json.tool
```

### Step 2 — Register with Claude Code

```bash
claude mcp add x64dbg \
  --scope user \
  --transport http \
  -- http://127.0.0.1:3000/mcp

claude mcp list | grep x64dbg
```

> x64dbg must be running on VM4 with the MCP plugin loaded before this MCP will connect. The `lab` function starts the tunnel automatically.

---

## 11c. WinDbg MCP Server

mcp-windbg provides crash dump analysis and kernel debugging via CDB. Runs on VM4, accessed via SSH tunnel on port 8000.

```bash
claude mcp add windbg \
  --scope user \
  --transport http \
  -- http://127.0.0.1:8000/mcp

claude mcp list | grep windbg
```

---

## 12. REMnux Analysis Tools

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
| `javap` | Disassemble `.class` bytecode to JVM instructions | built-in |
| `jadx` | Decompile JAR and Android APK — best for obfuscated JARs (`/usr/local/jadx/bin/jadx`) | pre-installed |
| `java` / `jar` | JDK tools for manifest inspection and class listing | built-in |

### Go Binary Analysis

| Tool | Description | Source |
|------|-------------|--------|
| `go version -m` | Extract build info and module dependencies | install |
| `blint` | Go binary capability detection — MITRE ATT&CK mapping | install |
| `nm` / `objdump` | Symbol extraction — Go symbols are rarely stripped | built-in |
| `strings` | Go binaries embed module paths unobfuscated | built-in |

---

## 13. SIFT Forensic Tools

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

## 13b. Windows Dynamic Analysis Tools

### Debuggers

| Tool | Description | Source |
|------|-------------|--------|
| `x64dbg` (64-bit) | User-mode Windows debugger — breakpoints, memory inspection, API tracing, unpacking | install |
| `x32dbg` (32-bit) | 32-bit companion for analyzing 32-bit malware | install |
| `WinDbg / CDB` | Kernel-mode debugger — crash dump analysis, driver debugging | pre-installed |

### MCP Plugins

| Tool | Description | Source |
|------|-------------|--------|
| `x64dbg-mcp plugin` | Exposes x64dbg over HTTP/JSON-RPC on port 3000 — 60+ tools for AI-driven debugging | install |
| `mcp-windbg` | Python wrapper around CDB — exposes WinDbg over HTTP on port 8000 | install |

### Network Simulation

| Tool | Description | Source |
|------|-------------|--------|
| `FakeNet-NG 3.5` | Intercepts and simulates DNS, HTTP, FTP, SMTP, IRC — prevents malware from reaching real C2 | install |

### Autostart Configuration

| Process | Port | Purpose | Autostart |
|---------|------|---------|-----------|
| `x64dbg.exe` (64-bit) | 3000 | User-mode debugging via MCP | ✅ Yes |
| `x32dbg.exe` (32-bit) | 3000 | 32-bit debugging via MCP | ✅ Yes |
| `mcp-windbg.exe` | 8000 | WinDbg/CDB via MCP | ✅ Yes |
| `fakenet.exe` | — | Network traffic simulation | ⚠️ Manual |

> Always start FakeNet-NG **before** detonating a sample. Always revert the VM4 snapshot **after** each analysis session.

---

## 14. MCP Tool Groups

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
| Ghidra | `[GHIDRA]` | `ghidra_load_sample`, `ghidra_run_analysis`, `ghidra_list_functions`, `ghidra_decompile`, `ghidra_list_imports`, `ghidra_list_strings`, `ghidra_detect_malware`, `ghidra_extract_iocs` | Deep reverse engineering |

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

### x64dbg MCP Tool Groups

| Group | Key Tools | Use when |
|-------|-----------|----------|
| Debug Control | `debug_init`, `debug_attach_pid`, `debug_run`, `debug_pause`, `debug_stop`, `debug_step_into`, `debug_step_over`, `debug_step_out`, `debug_run_to` | Starting, stopping and stepping |
| Breakpoints | `breakpoint_set`, `breakpoint_delete`, `breakpoint_set_condition`, `breakpoint_set_log`, `breakpoint_list` | Setting API hooks, conditional stops |
| Registers | `register_get`, `register_set`, `register_get_batch`, `register_list` | Inspecting CPU state |
| Memory | `memory_read`, `memory_write`, `memory_search`, `memory_enumerate`, `memory_get_info` | Reading/searching process memory |
| Disassembly | `disassembly_at`, `disassembly_range`, `disassembly_function`, `assembler_assemble` | Viewing and modifying code |
| Modules | `module_list`, `module_get_imports`, `module_get_exports` | Loaded DLLs and API imports |
| Dumping | `dump_memory_region`, `dump_module`, `dump_detect_oep`, `dump_analyze_module` | Unpacking — OEP detection and dump |
| Threads/Stack | `thread_list`, `stack_get_trace`, `stack_read_frame` | Multi-threaded malware analysis |
| Symbols | `symbol_list`, `symbol_search`, `symbol_set_label`, `symbol_set_comment` | Annotating functions during analysis |

---

## 15. Analysis Workflows

Run from `~/malware-analysis` using the `lab` command.

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

### Workflow 10 — Dynamic Analysis with x64dbg + FakeNet

```
Dynamic analysis of /home/remnux/samples/malware.exe:

Pre-flight (do manually on VM4 before starting):
- Start FakeNet: cd C:\Users\tasox\Downloads\fakenet3.5\fakenet3.5 && fakenet.exe
- Transfer sample: scp -i ~/.ssh/windows_key analyst@192.168.56.40:C:/samples/malware.exe

Then use x64dbg MCP to:
1. debug_init with C:\samples\malware.exe
2. breakpoint_set on suspicious APIs: CreateProcess, VirtualAlloc, WSAConnect, RegSetValue
3. debug_run to execute the sample
4. On each breakpoint hit: register_get_batch to capture CPU state
5. memory_read at addresses pointed to by registers
6. module_list to see what DLLs were loaded
7. stack_get_trace to see call chain
8. If packed: dump_detect_oep to find OEP then dump_module
9. Summarize: execution flow, APIs called, network IOCs from FakeNet, verdict
```

### Workflow 11 — Unpacking with x64dbg

```
Unpack the protected sample at C:\samples\packed.exe using x64dbg:

1. x64dbg debug_init with C:\samples\packed.exe
2. x64dbg debug_run — let the unpacker run
3. x64dbg dump_detect_oep to locate the Original Entry Point
4. x64dbg dump_module to extract the unpacked binary
5. Transfer the dump back to VM1:
   scp -i ~/.ssh/windows_key analyst@192.168.56.40:C:\dumps\unpacked.exe ~/downloads/
6. Transfer to REMnux: scp ~/downloads/unpacked.exe remnux@192.168.56.20:/home/remnux/samples/
7. remnux-ssh triage_full on the unpacked binary
8. remnux-ssh ghidra_load_sample for deep analysis
```

### Workflow 12 — Crash Dump Analysis with WinDbg

```
Analyze the crash dump at C:\dumps\crash.dmp using WinDbg MCP:

1. Use windbg MCP to open and analyze C:\dumps\crash.dmp
2. Get the exception record and faulting address
3. Get the call stack at time of crash
4. List loaded modules and check for suspicious ones
5. Check if any known malicious modules are present
6. Cross-reference any suspicious hashes with virustotal
7. Summarize: crash cause, faulting module, indicators of compromise
```

---

## 16. Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| SSH connection refused | SSH service not running or wrong IP | Run `sudo systemctl start ssh`. Verify IP with `ip a`. |
| MCP shows "Failed to connect" | Node.js path wrong or build not compiled | Run `npx tsc`. Re-register with `claude mcp remove` then `add`. |
| Ghidra returns "Unauthorized" | Auth token stripped through SSH layers | Use the ghidra-scripts wrapper approach — never pass token via `docker exec` args directly. |
| Ghidra container crashes | Binding 0.0.0.0 without auth token | Always set `GHIDRA_MCP_AUTH_TOKEN` when using `GHIDRA_MCP_BIND_ADDRESS=0.0.0.0`. |
| `ghidra_list_functions` returns empty | Sample loaded but not analyzed | Call `ghidra_run_analysis` with `entry_offset=0x0` after loading. Required for raw shellcode. |
| `ghidra_load_sample` fails on shellcode | Auto-detection fails on headerless binaries | Pass the `language` parameter: `x86:LE:64:default` for 64-bit, `x86:LE:32:default` for 32-bit. |
| `ghidra_load_sample` returns "File not found" | Container uses `/data/` not `/home/remnux/samples/` | The updated `ghidra-post.sh` translates paths automatically. If using `run_command` directly, use `/data/filename`. |
| MCP not visible in Claude session | Registered with project scope | Re-register with `--scope user`. Check `~/.claude.json` for duplicates. |
| npm install hangs on REMnux/SIFT | No internet (host-only network) | Re-enable NAT temporarily. Switch back to host-only after. |
| sift-ssh not connecting | Wrong username or key path | SIFT user is `sansforensics`. Test: `ssh -i ~/.ssh/sift_key sansforensics@192.168.56.30`. |
| Volatility3 returns no output | Wrong OS profile or image path | Try `vol -f image.dmp windows.info` first. Use full absolute path. |
| netplan apply fails on SIFT | nano not installed | Use `sudo tee /etc/netplan/...` instead. Check interface name with `ip a`. |
| x64dbg MCP "connection refused" | SSH tunnel not established or x64dbg not running | Ensure `lab` function was used to start tunnel. Verify x64dbg is open on VM4 with plugin loaded. |
| x64dbg MCP times out on debug_run | Sample exiting immediately or crashing | Check `debug_pause` first; use `breakpoint_set` on entry point before `debug_run`. |
| windbg MCP not responding | mcp-windbg not started or tunnel down | Check `C:\lab\startup.ps1` ran. Re-run tunnel: `ssh -N -L 8000:127.0.0.1:8000 windows`. |
| FakeNet not capturing traffic | FakeNet not started before sample | Always start FakeNet before `debug_init`. Check it's running with `ps` on VM4. |
| cfr/procyon empty output | Not a valid JAR or encrypted JAR | Verify with `file sample.jar`. Try `jadx` as fallback. |
| `go version -m` returns nothing | Stripped binary or not Go | Use `strings binary \| grep 'go[0-9]'`. Go binaries embed module paths even when stripped. |
| jadx not found | Not in PATH on REMnux | Use full path: `/usr/local/jadx/bin/jadx`. |

---

## 17. Security Notes

- [ ] **Never execute samples on VM1.** VM1 holds your API keys and MCP config. Samples only ever live on REMnux (`/home/remnux/samples/`), SIFT (`/cases/`), or Windows (`C:\samples\`).
- [ ] **Keep REMnux and SIFT host-only.** Switch back immediately after installing tools. Never leave analysis VMs on NAT during a session.
- [ ] **Use password-protected zips.** Always transfer samples as `zip -P infected sample.zip sample.exe`. This is the VT standard.
- [ ] **Revert snapshots between samples.** Take clean snapshots of REMnux and Windows before analysis. Revert after each session.
- [ ] **Always start FakeNet before detonating.** Never run samples on Windows without FakeNet active — malware may reach real C2 servers.
- [ ] **Disable shared clipboard and drag-and-drop** in VMware for REMnux and SIFT. Prevents accidental file transfer to the host.
- [ ] **API keys stay on VM1 only.** The VT API key is in `~/.config/malware-lab/secrets.env` on VM1. Never transfer to analysis VMs.
- [ ] **Hash all evidence.** Use `sift-ssh hash_file` before and after forensic analysis to maintain chain of custody.
- [ ] **The Ghidra auth token is lab-internal only.** `ghidra-lab-token` binds to `127.0.0.1` inside Docker — not exposed on the lab network.

---

*AI-Powered Malware Analysis Lab — Installation Guide v1.2*
*VM1: Ubuntu 24.04 · VM2: REMnux · VM3: SIFT · VM4: Windows 11*
