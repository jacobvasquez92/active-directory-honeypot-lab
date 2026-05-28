# 🛡️ Active Directory Honeypot Lab — VMware IDS & SIEM Monitoring

> **Blue Team Lab** | VMware Workstation · Snort IDS · Wazuh SIEM · Windows Server 2019 · Windows 10  
[Tutorial web page](https://jacobvasquez92.github.io/active-directory-honeypot-lab/)
---

A hands-on cybersecurity lab that deploys a domain-joined Windows honeypot with real-time intrusion detection and SIEM monitoring — built entirely in VMware Workstation. This project demonstrates network segmentation, IDS rule authoring, SIEM deployment, and end-to-end attack chain detection relevant to SOC Analyst and Blue Team roles.

---
## 🗺️ General Lab Architecture
<img width="761" height="613" alt="image" src="https://github.com/user-attachments/assets/ebb9439c-0ce1-4d3a-be0f-59c8db26dfea" />

## 🗺️ Lab Specific Architecture 

```
┌──────────────────────────────────────────────────────────────┐
│                    VMware Workstation                        │
│                                                              │
│  ┌─────────────────┐    ┌─────────────────┐    ┌──────────┐ │
│  │  DC-Win2019     │    │   CLIENT1        │    │  Wazuh   │ │
│  │  VMnet1         │    │  ⚠ HONEYPOT ⚠   │    │  Ubuntu  │ │
│  │  192.168.1.10   │    │                  │    │          │ │
│  │  • AD DS        │    │ NIC1: VMnet1     │    │ VMnet1   │ │
│  │  • DNS          │    │ NIC2: Bridged ←──┼────┼─Internet │ │
│  │  • DHCP         │    │ NIC3: VMnet3     │    │ 172.16.  │ │
│  └─────────────────┘    │ • Snort IDS      │    │ 0.101    │ │
│                         │ • Wazuh Agent    │    │ • SIEM   │ │
│                         │ • Sysmon         │    │ Dashboard│ │
│                         └─────────────────┘    └──────────┘ │
└──────────────────────────────────────────────────────────────┘

VMnet1 = Internal domain network (Host-only)
VMnet3 = Log forwarding channel (Host-only, isolated)
Bridged = Internet-facing attack surface (router port forwarding)
```

| Traffic Path | Status | Enforced By |
|---|---|---|
| Internet → CLIENT1 NIC2 | ✅ Allowed | Router port forwarding |
| CLIENT1 ↔ DC (VMnet1) | ✅ Allowed | Domain membership |
| CLIENT1 VMnet3 → VMnet1 | ❌ Blocked | IP forwarding disabled + firewall |
| VMnet3 → Host OS | ❌ Blocked | VMware virtual switch isolation |
| Wazuh Agent → Manager (1514) | ✅ Allowed | VMnet1 only |

---

## 📋 Prerequisites

- VMware Workstation Pro (or Player)
- Windows Server 2019 ISO (DC)
- Windows 10 ISO (CLIENT1)
- Ubuntu 22.04 LTS ISO (Wazuh Manager)
- Minimum host RAM: 16GB recommended (4GB DC + 4GB CLIENT1 + 4GB Wazuh + host)

> ⚠️ **Before starting Phase 1:** Take a VMware snapshot of CLIENT1 labeled `pre-honeypot-baseline`

---

## 🔴 Phase 1 — Isolating CLIENT1 as a Honeypot

### Step 1 — Create VMnet3

1. VMware → **Edit → Virtual Network Editor**
2. **Add Network** → VMnet3 → **Host-only**
3. **Disable DHCP** on VMnet3
4. Uncheck "Connect a host virtual adapter to this network"

### Step 2 — Add Three NICs to CLIENT1

Shut down CLIENT1 → VM Settings → Add → Network Adapter (add twice):

| NIC | Network | IP | Purpose |
|---|---|---|---|
| NIC 1 | VMnet1 | DHCP from DC | Domain membership |
| NIC 2 | Bridged | DHCP reservation on router | Internet-facing honeypot |
| NIC 3 | VMnet3 | Static: 192.168.100.10 | Wazuh log channel |

> 💡 **NIC 2 tip:** Set a DHCP reservation in your router binding CLIENT1's MAC to a fixed IP (e.g. 192.168.1.150), then port-forward 3389, 445, 21, and 23 to that IP.

### Step 3 — Enforce Network Isolation (PowerShell as Admin on CLIENT1)

```powershell
# Set static IP on VMnet3 with NO gateway
netsh interface ip set address name="Ethernet 3" static 192.168.100.10 255.255.255.0

# Disable IP forwarding — prevents CLIENT1 routing between adapters
Set-NetIPInterface -Forwarding Disabled -AddressFamily IPv4

# Block cross-network traffic via firewall
New-NetFirewallRule -DisplayName "Block VMnet3 to VMnet1" `
  -Direction Outbound -RemoteAddress 192.168.1.0/24 `
  -InterfaceAlias "Ethernet 3" -Action Block -Profile Any

New-NetFirewallRule -DisplayName "Block VMnet1 to VMnet3" `
  -Direction Outbound -RemoteAddress 192.168.100.0/24 `
  -InterfaceAlias "Ethernet 1" -Action Block -Profile Any
```

```powershell
# Run on DC — block VMnet3 subnet inbound
New-NetFirewallRule -DisplayName "Block Honeypot VMnet3" `
  -Direction Inbound -RemoteAddress 192.168.100.0/24 `
  -Action Block -Profile Any
```

**Verify isolation:**
```powershell
# Should FAIL
Test-NetConnection -ComputerName 192.168.1.1 -Source 192.168.100.10

# Should SUCCEED
Test-NetConnection -ComputerName 172.16.0.101 -Port 1514
```

### Step 4 — Intentionally Weaken CLIENT1

> ⚠️ **Snapshot first!** Right-click CLIENT1 → Snapshot → `pre-honeypot-baseline`

```powershell
# 1. Disable Windows Defender
Set-MpPreference -DisableRealtimeMonitoring $true
Set-MpPreference -DisableBehaviorMonitoring $true
Set-MpPreference -DisableIOAVProtection $true

# 2. Disable Windows Firewall
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

# 3. Enable SMBv1 (Windows 10 — feature must be installed first)
Enable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol -All -NoRestart
Enable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol-Server -NoRestart
# Reboot, then:
Set-SmbServerConfiguration -EnableSMB1Protocol $true -Force

# 4. Enable RDP without NLA
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
  -Name "fDenyTSConnections" -Value 0
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
  -Name "UserAuthentication" -Value 0

# 5. Create honeypot users
$pw1 = ConvertTo-SecureString "admin123" -AsPlainText -Force
New-LocalUser -Name "admin" -Password $pw1 -FullName "Administrator"
Add-LocalGroupMember -Group "Administrators" -Member "admin"

$pw2 = ConvertTo-SecureString "Password1" -AsPlainText -Force
New-LocalUser -Name "svc_backup" -Password $pw2 -Description "Backup Service Account"
Add-LocalGroupMember -Group "Administrators" -Member "svc_backup"

# 6. Install FTP (Windows 10 uses DISM, not Install-WindowsFeature)
dism /online /Enable-Feature /FeatureName:IIS-WebServer /All
dism /online /Enable-Feature /FeatureName:IIS-FTPServer /All
dism /online /Enable-Feature /FeatureName:IIS-ManagementConsole /All
# Then configure via IIS Manager → Add FTP Site → C:\shares, port 21, anonymous auth

# 7. Enable Telnet
dism /online /Enable-Feature /FeatureName:TelnetServer
net start tlntsvr

# 8. Create lure files
New-Item -Path "C:\shares" -ItemType Directory -Force
New-SmbShare -Name "shared" -Path "C:\shares" -FullAccess "Everyone"
Set-Content "C:\shares\passwords.txt" "admin:admin123`nroot:toor`nbackup:backup2024"
Set-Content "C:\shares\network_diagram.txt" "DC: 192.168.1.10`nSQL: 192.168.1.20"
```

**Verify honeypot status:**
```powershell
Write-Host "=== Honeypot Status ===" -ForegroundColor Cyan
Get-MpPreference | Select-Object DisableRealtimeMonitoring
Get-NetFirewallProfile | Select-Object Name, Enabled
Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol
Get-LocalUser | Select-Object Name, Enabled
netstat -an | findstr "LISTENING"
```

---

## 🟡 Phase 2 — Snort IDS on CLIENT1

### Step 1 — Install Npcap
1. Download from https://npcap.com/#download
2. Install with ✅ **WinPcap API-compatible mode** checked
3. Reboot CLIENT1

### Step 2 — Install Snort 2.9.x
1. Download **Snort 2.9.x Windows binary** from https://www.snort.org/downloads (not Snort 3)
2. Install to `C:\Snort` (accept defaults)

### Step 3 — Community Rules & Support Files
```cmd
REM Download community.rules from snort.org/downloads#rules
REM Copy to C:\Snort\rules\community.rules

type nul > C:\Snort\rules\local.rules
type nul > C:\Snort\etc\white_list.rules
type nul > C:\Snort\etc\black_list.rules

mkdir C:\Snort\lib\snort_dynamicpreprocessor
mkdir C:\Snort\lib\snort_dynamicengine
mkdir C:\Snort\lib\snort_dynamicrules
```

### Step 4 — Configure snort.conf

Open `C:\Snort\etc\snort.conf` as Administrator and set:

```conf
# Network variables (~line 45)
ipvar HOME_NET 192.168.1.0/24
ipvar EXTERNAL_NET !$HOME_NET

# Rule paths (~line 104)
var RULE_PATH C:\Snort\rules
var SO_RULE_PATH C:\Snort\rules
var PREPROC_RULE_PATH C:\Snort\rules
var WHITE_LIST_PATH C:\Snort\etc
var BLACK_LIST_PATH C:\Snort\etc
```

**Fix Linux-style paths** (Ctrl+H in Notepad):

| Find | Replace |
|---|---|
| `/usr/local/lib/snort_dynamicpreprocessor/` | `C:\Snort\lib\snort_dynamicpreprocessor` |
| `/usr/local/lib/snort_dynamicengine/` | `C:\Snort\lib\snort_dynamicengine` |

**Comment out all rule includes except yours:**
- Ctrl+H → Find: `include $RULE_PATH/` → Replace: `# include $RULE_PATH/`
- Remove `#` from only `community.rules` and `local.rules`
- Add `#` in front of any bare text lines like `dynamic library rules`

### Step 5 — Custom Honeypot Rules (local.rules)

```snort
# RDP brute force
alert tcp any any -> $HOME_NET 3389 (msg:"RDP Brute Force Attempt"; threshold:type threshold, track by_src, count 5, seconds 60; sid:1000001; rev:1;)

# SMB scan
alert tcp any any -> $HOME_NET 445 (msg:"SMB Scan Detected"; flags:S; threshold:type threshold, track by_src, count 10, seconds 10; sid:1000002; rev:1;)

# ICMP ping
alert icmp any any -> $HOME_NET any (msg:"ICMP Ping Test"; sid:1000003; rev:1;)

# Telnet
alert tcp any any -> $HOME_NET 23 (msg:"Telnet Connection Attempt"; sid:1000004; rev:1;)

# FTP
alert tcp any any -> $HOME_NET 21 (msg:"FTP Connection Attempt"; sid:1000005; rev:1;)
```

> ⚠️ Ensure all quotes are **straight quotes** `"` not curly/smart quotes — copy-paste from browsers can corrupt them.

### Step 6 — Find Interface & Test

```cmd
REM List interfaces — note index of your Bridged adapter
C:\Snort\bin\snort.exe -W

REM Test config (no traffic captured)
C:\Snort\bin\snort.exe -T -i 2 -c C:\Snort\etc\snort.conf
```
Expected: `Snort successfully validated the configuration!`

### Step 7 — Run Snort

```cmd
C:\Snort\bin\snort.exe -i 2 -c C:\Snort\etc\snort.conf -l C:\Snort\log -A full -K ascii
```

Run as background process (if service install fails):
```cmd
start /min "Snort IDS" "C:\Snort\bin\snort.exe" -i 2 -c "C:\Snort\etc\snort.conf" -l "C:\Snort\log" -A full -K ascii
```

Test by pinging CLIENT1's bridged IP, then check `C:\Snort\log\alert.ids`.

---

## 🟢 Phase 3 — Wazuh SIEM

### Step 1 — Deploy Wazuh Manager VM

- Ubuntu 22.04 LTS · 4GB RAM · 2 vCPUs · 50GB disk · NIC on VMnet1

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
# If OS compatibility error, add -i flag to bypass check
```

Retrieve credentials after install:
```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

Access dashboard: `https://172.16.0.101` — Username: `admin`, Password: from above output.

### Step 2 — Install Wazuh Agent on CLIENT1

```powershell
# Download (must match manager version)
$client = New-Object System.Net.WebClient
$client.DownloadFile(
  "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi",
  "C:\Users\admin\Downloads\wazuh-agent.msi"
)

# Verify ~50MB
Get-Item "C:\Users\admin\Downloads\wazuh-agent.msi" | Select-Object Name, Length

# Install
msiexec.exe /i "C:\Users\admin\Downloads\wazuh-agent.msi" `
  WAZUH_MANAGER="172.16.0.101" `
  WAZUH_AGENT_NAME="CLIENT1-Honeypot" `
  /l*v "C:\wazuh-install.log"

NET START WazuhSvc

# Verify connection
Test-NetConnection -ComputerName 172.16.0.101 -Port 1514
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 20
```

### Step 3 — Install Agent on DC

```powershell
msiexec.exe /i "C:\Users\admin\Downloads\wazuh-agent.msi" `
  WAZUH_MANAGER="172.16.0.101" `
  WAZUH_AGENT_NAME="DC-DomainController" `
  /l*v "C:\wazuh-install-dc.log"
NET START WazuhSvc
```

### Step 4 — Ingest Snort Alerts into Wazuh

Open `C:\Program Files (x86)\ossec-agent\ossec.conf` in Notepad **as Administrator**.

Find `</ossec_config>` and insert immediately before it:

```xml
  <localfile>
    <log_format>snort-full</log_format>
    <location>C:\Snort\log\alert.ids</location>
  </localfile>

</ossec_config>
```

Restart the agent:
```powershell
Restart-Service WazuhSvc
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 30
```

### Step 5 — Install Sysmon for Deep Visibility

```powershell
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile sysmon.zip
Expand-Archive sysmon.zip -DestinationPath C:\Sysmon

Invoke-WebRequest -Uri "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml" `
  -OutFile C:\Sysmon\sysmon-config.xml

C:\Sysmon\Sysmon64.exe -accepteula -i C:\Sysmon\sysmon-config.xml
```

---

## ✅ Verification

In the Wazuh Dashboard (`https://172.16.0.101`):

- **Agents** → CLIENT1-Honeypot and DC-DomainController show 🟢 Active
- **Security Events** → Snort alerts appear with source IP and timestamps
- **Integrity Monitoring** → Sysmon process events

**Attack signatures this lab captures:**

| Technique | Detection Source | Event |
|---|---|---|
| RDP/SMB brute-force | Snort + Windows Events | Rule 1000001 + Event ID 4625 |
| Credential access (Mimikatz) | Sysmon | Event ID 10 (lsass access) |
| Port scanning | Snort | Rule 1000002 SYN flood |
| Lateral movement | Sysmon + Wazuh | Unusual process spawning |
| Persistence | Sysmon | Registry run key / scheduled task |

---

## 🔧 Troubleshooting

See the dedicated **[Troubleshooting Guide](troubleshooting.md)** for documented issues and resolutions encountered during this lab, including:

- [TS-01] NIC 2 bridged adapter and DHCP reservation setup
- [TS-02] Snort dynamic preprocessor Linux path errors on Windows
- [TS-03] SMBv1 "service does not exist" error on Windows 10
- [TS-04] WHITE_LIST_PATH double-definition causing path concatenation
- [TS-05] "Unknown rule type: dynamic" — uncommented section headers
- [TS-06] Install-WindowsFeature not recognized on Windows 10
- [TS-07] Snort ICMP alerts not firing due to threshold count
- [TS-08] Snort Windows service registration failure
- [TS-09] Wazuh OS compatibility check failure
- [TS-10] Wazuh dashboard credentials location
- [TS-11] Wazuh MSI corrupt/incomplete download
- [TS-12] WazuhSvc "service name is invalid" error
- [TS-13] Editing ossec.conf step-by-step

---

## 🛠️ Skills Demonstrated

| Skill | Evidence |
|---|---|
| **Network Segmentation** | Multi-VMnet topology isolating honeypot from internal domain |
| **Active Directory** | Domain-joined honeypot with GPO, DHCP/DNS from DC |
| **IDS Configuration** | Snort 2.9.x installation, rule authoring, config troubleshooting |
| **SIEM Deployment** | Wazuh all-in-one on Ubuntu, multi-agent enrollment, cross-source ingestion |
| **PowerShell** | System config, firewall rules, service management, connectivity testing |
| **Threat Detection** | Coverage across brute-force, LSASS access, lateral movement, persistence |

---

## 🤳 Connect

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Jacob%20Vasquez-blue?style=flat&logo=linkedin)](https://linkedin.com/in/jacob-vasquez-b46056257)
