# triage-llm-lab

> AI-powered malware analysis and digital forensics lab — Claude Code orchestrating REMnux, SIFT Workstation, Ghidra, and VirusTotal over custom MCP servers in an isolated multi-VM environment.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Host Machine (macOS / Linux)                                            │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  VMware Host-Only Network: 192.168.56.0/24                         │ │
│  │                                                                    │ │
│  │  ┌──────────────────┐    ┌──────────────────┐    ┌─────────────┐  │ │
│  │  │  VM1 — Ubuntu    │    │  VM2 — REMnux    │    │  VM3 — SIFT │  │ │
│  │  │  192.168.56.10   │    │  192.168.56.20   │    │ 192.168.56  │  │ │
│  │  │                  │    │                  │    │    .30      │  │ │
│  │  │  Claude Code     │◄──►│  Analysis Tools  │    │  Forensics  │  │ │
│  │  │  MCP Servers     │    │  Ghidra (Docker) │    │  Volatility │  │ │
│  │  │  VT API Key      │    │  Host-only NIC   │◄──►│  Sleuthkit  │  │ │
│  │  └──────────────────┘    └──────────────────┘    └─────────────┘  │ │
│  │          │                        │                                │ │
│  │    remnux-ssh MCP           SSH tunnel                             │ │
│  │    sift-ssh MCP             (port 8089)                            │ │
│  │    virustotal MCP                                                  │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  VM4 — Windows 10 (192.168.56.40) — future dynamic analysis             │
└─────────────────────────────────────────────────────────────────────────┘
```

**VM roles:**

| VM | OS | IP | Role | Status |
|----|----|----|------|--------|
| VM1 | Ubuntu 24.04 | 192.168.56.10 | LLM host — Claude Code, MCP servers, API keys | Active |
| VM2 | REMnux | 192.168.56.20 | Static analysis — malware tools, Ghidra Docker | Active |
| VM3 | SIFT Workstation | 192.168.56.30 | DFIR — memory forensics, disk forensics, timelines | Active |
| VM4 | Windows 10 | 192.168.56.40 | Dynamic analysis (sandbox) | Future |

---

## Prerequisites

### Hardware

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 16 GB | 32 GB |
| Storage | 150 GB | 300 GB |
| CPU cores | 8 | 12+ |

### Software

- VMware Fusion (macOS) or VMware Workstation (Linux/Windows)
- [REMnux OVA](https://remnux.org/) — import as VM2
- [SIFT Workstation OVA](https://www.sans.org/tools/sift-workstation/) — import as VM3
- Node.js ≥ 20 (on VM1)
- TypeScript (`npm install -g typescript`) (on VM1)
- Claude Code CLI (on VM1)
- VirusTotal API key (free tier works for most tools)

---

## VM Setup

### VM1 — Ubuntu 24.04 (LLM Host)

```bash
# Update and install essentials
sudo apt update && sudo apt upgrade -y
sudo apt install -y openssh-server curl git build-essential

# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install TypeScript globally
npm install -g typescript

# Install Claude Code CLI
npm install -g @anthropic/claude-code

# Create working directory
mkdir -p ~/malware-analysis ~/downloads
```

**Static IP configuration** (`/etc/netplan/01-netcfg.yaml`):

```yaml
network:
  version: 2
  ethernets:
    ens33:          # NAT adapter — internet access
      dhcp4: true
    ens38:          # Host-only adapter
      dhcp4: no
      addresses:
        - 192.168.56.10/24
```

```bash
sudo netplan apply
```

---

### VM2 — REMnux (Static Analysis)

```bash
# Enable SSH server
sudo systemctl enable ssh --now

# Set static IP (edit /etc/netplan/... for host-only NIC)
# Host-only NIC: 192.168.56.20/24

# Verify core tools
file --version && strings --version && xxd --version
die --version    # Detect-It-Easy
floss --version
capa --version
```

**SSH key setup from VM1:**

```bash
# On VM1 — generate dedicated key pair
ssh-keygen -t ed25519 -f ~/.ssh/remnux_key -C "remnux-mcp" -N ""

# Copy public key to REMnux
ssh-copy-id -i ~/.ssh/remnux_key.pub remnux@192.168.56.20

# Test
ssh -i ~/.ssh/remnux_key remnux@192.168.56.20 "uname -a"
```

---

### VM3 — SIFT Workstation (Digital Forensics)

**Default credentials:** `sansforensics` / `forensics`

```bash
# On SIFT — enable copy/paste (VMware tools)
sudo apt install -y open-vm-tools open-vm-tools-desktop

# Enable SSH server
sudo systemctl enable ssh --now

# Verify core forensic tools
vol -h 2>/dev/null | head -3       # Volatility3
fls --version                       # Sleuthkit
log2timeline.py --version           # Plaso
autopsy --version 2>/dev/null || echo "check /opt/autopsy"
bulk_extractor --version
foremost -V
rip.pl --help 2>&1 | head -3
dcfldd --version
ssdeep -V

# Install additional tools
sudo apt install -y sleuthkit yara python3-yara dcfldd hashdeep ssdeep \
  exiftool libewf2t64 afflib-tools python3-registry
pip3 install construct dissect.cstruct --break-system-packages

# Create cases directory structure
sudo mkdir -p /cases/carved /cases/dumps /cases/recovered
sudo chown -R sansforensics:sansforensics /cases
```

**Static IP configuration** (`/etc/netplan/01-netcfg.yaml`):

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
    ens38:          # Host-only adapter
      dhcp4: no
      addresses:
        - 192.168.56.30/24
```

```bash
sudo netplan apply
```

**SSH key setup from VM1:**

```bash
# On VM1 — generate dedicated key for SIFT
ssh-keygen -t ed25519 -f ~/.ssh/sift_key -C "sift-mcp" -N ""

# Copy to SIFT
ssh-copy-id -i ~/.ssh/sift_key.pub sansforensics@192.168.56.30

# Test
ssh -i ~/.ssh/sift_key sansforensics@192.168.56.30 "uname -a"
```

---

## MCP Servers

### remnux-ssh MCP

Gives Claude direct access to REMnux analysis tools over SSH.

```bash
# On VM1 — create project
mkdir -p ~/mcp/remnux-mcp/src
cd ~/mcp/remnux-mcp

npm init -y
npm install @modelcontextprotocol/sdk ssh2 zod
npm install -D typescript @types/node @types/ssh2
npm pkg set type="module"

# Create tsconfig.json
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

# Place src/index.ts, then build
npx tsc
```

**Register with Claude Code:**

```bash
claude mcp add remnux-ssh \
  --scope user \
  -- node /home/$USER/mcp/remnux-mcp/build/index.js \
  --host 192.168.56.20 \
  --username remnux \
  --private-key /home/$USER/.ssh/remnux_key

# Verify
claude mcp list | grep remnux
```

---

### Ghidra MCP (Docker on REMnux)

Headless Ghidra for deep binary reverse engineering, accessed over SSH tunnel.

**Step 1 — Build Docker image on VM1:**

```bash
# On VM1
mkdir -p ~/ghidra-mcp-build
cd ~/ghidra-mcp-build

# Build the headless Ghidra MCP image
docker build -t ghidra-mcp-headless:latest .

# Export as tar for transfer to REMnux
docker save ghidra-mcp-headless:latest -o ghidra-mcp-headless.tar
```

**Step 2 — Transfer image to REMnux:**

```bash
scp -i ~/.ssh/remnux_key ghidra-mcp-headless.tar remnux@192.168.56.20:/home/remnux/
```

**Step 3 — Load and run on REMnux:**

```bash
# On VM2 — REMnux
docker load -i /home/remnux/ghidra-mcp-headless.tar

# Create run script
python3 -c "
import os
script = '''#!/bin/bash
docker run -d --name ghidra-mcp --memory 2g \\
  -p 127.0.0.1:8089:8089 \\
  -v /home/remnux/samples:/data \\
  -v ghidra-projects:/projects \\
  -e GHIDRA_MCP_PORT=8089 \\
  -e GHIDRA_MCP_BIND_ADDRESS=0.0.0.0 \\
  -e GHIDRA_MCP_AUTH_TOKEN=ghidra-lab-token \\
  -e JAVA_OPTS=-Xmx2g \\
  ghidra-mcp-headless:latest
'''
with open('/home/remnux/run-ghidra-mcp.sh', 'w') as f:
    f.write(script)
os.chmod('/home/remnux/run-ghidra-mcp.sh', 0o755)
print('Script written')
"

# Start container
/home/remnux/run-ghidra-mcp.sh

# Verify (~20s startup)
sleep 20
docker exec ghidra-mcp curl -s http://localhost:8089/get_version
```

Expected: `{"plugin_version": "5.13.1-headless", "plugin_name": "GhidraMCP Headless", "mode": "headless"}`

---

### sift-ssh MCP

Gives Claude direct access to SIFT forensic tools over SSH — same pattern as remnux-ssh.

```bash
# On VM1
mkdir -p ~/mcp/sift-mcp/src
cd ~/mcp/sift-mcp

npm init -y
npm install @modelcontextprotocol/sdk ssh2 zod
npm install -D typescript @types/node @types/ssh2
npm pkg set type="module"

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

# Place src/index.ts, then build
npx tsc
```

**Register with Claude Code:**

```bash
claude mcp add sift-ssh \
  --scope user \
  -- node /home/$USER/mcp/sift-mcp/build/index.js \
  --host 192.168.56.30 \
  --username sansforensics \
  --private-key /home/$USER/.ssh/sift_key

# Verify
claude mcp list | grep sift
```

**Test:**

```
Use sift-ssh run_command to run: uname -a && vol -h 2>/dev/null | head -3
```

---

### virustotal MCP

```bash
# On VM1 — store API key
mkdir -p ~/.config/malware-lab
echo "VT_API_KEY=your_key_here" > ~/.config/malware-lab/secrets.env

# Register virustotal MCP
claude mcp add virustotal \
  --scope user \
  --env VT_API_KEY \
  -- node /home/$USER/mcp/virustotal-mcp/build/index.js
```

---

## REMnux Analysis Tools

### Static Analysis

| Tool | Description |
|------|-------------|
| `file` | Identify file type by magic bytes |
| `strings` | Extract printable strings |
| `xxd` | Hex dump files |
| `diec` / `die` | Detect packers, compilers, protectors |
| `FLOSS` | Extract obfuscated + stack strings |
| `CAPA` | MITRE ATT&CK capability detection |
| `pescanner` | PE header analysis |
| `exiftool` | File metadata extraction |
| `binwalk` | Embedded file extraction |
| `upx` | UPX packer detection / unpacking |

### Office Document Analysis

| Tool | Description |
|------|-------------|
| `olevba` | VBA macro extraction and analysis |
| `oleid` | OLE file indicator detection |
| `oledump` | OLE stream dumping |
| `mraptor` | Malicious macro detection |
| `ViperMonkey` | VBA macro emulation |

### Script Deobfuscation

| Tool | Description |
|------|-------------|
| `synchrony` | JavaScript deobfuscation (obfuscator.io) |
| `js-beautify` | JavaScript formatting |

### ELF / Linux Analysis

| Tool | Description |
|------|-------------|
| `readelf` | ELF header and section analysis |
| `objdump` | Disassembly and symbol inspection |
| `nm` | Symbol table listing |
| `ldd` | Shared library dependencies |
| `strace` / `ltrace` | System and library call tracing |

### Python Malware

| Tool | Description |
|------|-------------|
| `pyinstxtractor-ng` | Unpack PyInstaller bundles |
| `uncompyle6` | Decompile Python .pyc bytecode |

### Windows Installer

| Tool | Description |
|------|-------------|
| `msiextract` | Extract MSI file contents |
| `msidump` | Dump MSI database tables |
| `7z` / `unzip` | Extract MSIX/AppX packages |
| `cabextract` | Extract Cabinet files |

### .NET Assembly Analysis

| Tool | Description |
|------|-------------|
| `ilspycmd` | Decompile .NET assemblies to C# source |
| `monodis` | Disassemble .NET IL bytecode, extract user strings |
| `pedump` | PE/CLI header and metadata table analysis |
| `dnfile` | Python library for .NET metadata parsing |
| `pefile` | Python PE structure parsing library |

```bash
# Install .NET Python libraries (REMnux, NAT enabled temporarily)
pip3 install dnfile pefile --break-system-packages
python3 -c "import dnfile, pefile; print('dnfile', dnfile.__version__, 'pefile ok')"
```

---

## MCP Tool Groups

### remnux-ssh Tool Groups

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
| .NET Assembly | `[DOTNET]` | `dotnet_decompile`, `dotnet_il_disasm`, `dotnet_pe_headers`, `dotnet_metadata`, `dotnet_strings`, `dotnet_resources`, `triage_dotnet` | C#, VB.NET, F# assemblies |

### sift-ssh Tool Groups

| Group | Tag | Tools | Use when |
|-------|-----|-------|----------|
| Utility | `[UTILITY]` | `run_command`, `list_cases`, `hash_file` | Case management, hashing evidence |
| Memory Forensics | `[MEMORY]` | `memory_info`, `memory_pslist`, `memory_netscan`, `memory_malfind`, `memory_cmdline`, `memory_dlllist`, `memory_handles`, `memory_registry`, `memory_dump_process`, `triage_memory` | Windows/Linux memory dumps (.dmp, .raw) |
| Disk Forensics | `[DISK]` | `disk_info`, `disk_list_files`, `disk_recover_files`, `disk_extract_file`, `triage_disk` | Disk images (.dd, .E01) |
| Timeline Analysis | `[TIMELINE]` | `timeline_create`, `timeline_filter`, `timeline_search` | Super-timeline creation and investigation |
| Artifact Extraction | `[ARTIFACT]` | `extract_artifacts`, `carve_files`, `forensic_copy` | Bulk extraction, file carving, forensic imaging |
| Registry Forensics | `[REGISTRY]` | `registry_analyze`, `registry_autoruns`, `registry_useractivity` | Windows registry hive analysis |
| Triage | `[TRIAGE]` | `triage_case`, `triage_memory`, `triage_disk` | First look at any new evidence file |

### VirusTotal MCP Tools

| Tool | Input | Description | Tier |
|------|-------|-------------|------|
| `get_file_report` | MD5 / SHA1 / SHA256 | Full detection report — AV verdicts, sandbox scores, metadata | Free |
| `get_file_behavior` | SHA256 | Dynamic analysis — network connections, registry changes, dropped files | Free |
| `get_file_relationship` | SHA256 + relationship | Related files, dropped files, contacted IPs/domains, execution parents | Free |
| `get_url_report` | URL | URL reputation, category, detections | Free |
| `get_ip_report` | IP address | IP reputation, ASN, country, associated files and URLs | Free |
| `get_domain_report` | Domain | Domain reputation, WHOIS, DNS history, associated malware | Free |
| `search` | VTI query string | Advanced search — e.g. `type:peexe tag:signed` | Free |
| `download_file` | SHA256 + output path | Download as password-protected zip (password: `infected`) to VM1 | Premium |

> `download_file` saves to `~/downloads/` on VM1. Always transfer to REMnux via `scp` before unzipping — never extract samples on VM1.

---

## Analysis Workflows

Run these Claude Code prompts from `~/malware-analysis` on VM1.

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

### Workflow 6 — Memory Forensics

```
Analyze the memory dump at /cases/memory.dmp on SIFT:

1. Use sift-ssh triage_memory on /cases/memory.dmp
2. Identify any suspicious processes from pslist (unknown names, unusual paths)
3. Use sift-ssh memory_malfind to find injected code
4. For any suspicious PIDs found, use sift-ssh memory_cmdline and memory_dlllist
5. Use sift-ssh memory_netscan to identify C2 connections
6. Cross-reference any IPs/domains with virustotal
7. Use sift-ssh memory_dump_process to extract suspicious process for static analysis
8. Summarize: compromised processes, IOCs, attacker activity
```

### Workflow 7 — Disk Image Forensics

```
Investigate the disk image at /cases/disk.dd on SIFT:

1. Use sift-ssh hash_file to verify evidence integrity
2. Use sift-ssh triage_disk to get partition layout and filesystem info
3. Use sift-ssh disk_list_files to browse the file system
4. Use sift-ssh timeline_create to build a super-timeline
5. Use sift-ssh timeline_search to find activity around the incident timeframe
6. Use sift-ssh carve_files to recover deleted executables
7. Use sift-ssh extract_artifacts for bulk IOC extraction
8. Transfer any carved executables to REMnux for static analysis
9. Summarize: timeline of events, suspicious files, IOCs
```

### The `lab` Shortcut

Add to `~/.bashrc` on VM1 to start Claude with all tools listed at startup:

```bash
lab() {
  cd ~/malware-analysis
  echo "Loading malware analysis lab..."
  claude "Show all available tools for this malware analysis lab grouped by category:
1. List all tools from remnux-ssh MCP grouped by their [CATEGORY] tag
2. List all tools from sift-ssh MCP grouped by their [CATEGORY] tag
3. List all tools from virustotal MCP with a short description of each
After showing the full list, wait for my next instruction."
}
```

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| SSH connection refused | SSH service not running or wrong IP | Run `sudo systemctl start ssh` on REMnux/SIFT. Verify IP with `ip a`. |
| MCP shows "Failed to connect" | Node.js path wrong or build not compiled | Run `npx tsc` in `~/mcp/remnux-mcp`. Re-register with `claude mcp remove` then `add`. |
| Ghidra returns "Unauthorized" | Auth token not passed correctly through SSH layers | Use the ghidra-scripts wrapper approach — do not pass token directly via `docker exec`. |
| Ghidra container crashes on start | Binding to `0.0.0.0` without auth token | Always set `GHIDRA_MCP_AUTH_TOKEN` when using `GHIDRA_MCP_BIND_ADDRESS=0.0.0.0`. |
| MCP not visible in Claude session | Registered with project scope, not user scope | Re-register with `--scope user` flag. Check `~/.claude.json` for duplicate entries. |
| SSH tunnel lost after reboot | Tunnel not persistent | Add to `~/.bashrc`: `ssh -i ~/.ssh/remnux_key -N -L 8089:127.0.0.1:8089 remnux@192.168.56.20 &` |
| npm install hangs on REMnux | Registry blocked (no internet) | Install on VM1, then `scp node_modules` to REMnux. Or temporarily enable NAT. |
| sift-ssh MCP not connecting | Wrong username or key path | SIFT default user is `sansforensics` not `ubuntu`. Verify: `ssh -i ~/.ssh/sift_key sansforensics@192.168.56.30` |
| Volatility3 returns no output | Wrong OS profile or memory image path | Try `vol -f image.dmp windows.info` first to detect OS. Use full absolute path. |
| netplan apply fails on SIFT | Syntax error or nano not installed | Use `sudo tee /etc/netplan/...` instead of nano. Check interface name with `ip a` first. |

---

## Security Notes

- **Never execute samples on VM1.** VM1 holds your API keys and MCP config. Samples only ever live on REMnux (`/home/remnux/samples/`).
- **Keep REMnux host-only.** Switch back to host-only immediately after installing tools. A single NAT session for installs is fine; leave it off during analysis.
- **Use password-protected zips.** Always transfer as `zip -P infected sample.zip sample.exe`. This is the VirusTotal standard.
- **Revert snapshots between samples.** Take a clean snapshot of REMnux before analysis; revert after each session.
- **Disable shared clipboard and drag-and-drop** in VMware settings for REMnux. This prevents accidental file transfer to the host.
- **API keys stay on VM1 only.** The VirusTotal API key is stored in `~/.config/malware-lab/secrets.env` on VM1 — never transferred to REMnux.
- **The Ghidra auth token is lab-internal only.** `ghidra-lab-token` is bound to `127.0.0.1` inside Docker — not exposed on the network.

---

## License

MIT — for authorized security research and education only.
