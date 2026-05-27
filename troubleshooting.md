# 🔧 Troubleshooting Guide — Active Directory Honeypot Lab

> Cross-referenced with the [main tutorial README](README.md). Each issue below documents an actual problem encountered during this lab build, with the exact error and resolution.

---

## Quick Reference Index

| ID | Issue | Phase |
|---|---|---|
| [TS-01](#ts-01) | NIC 2 bridged adapter — where does the IP come from? | Phase 1, Step 2 |
| [TS-02](#ts-02) | Snort ERROR: Could not stat dynamic module path `/usr/local/lib/...` | Phase 2, Step 4 |
| [TS-03](#ts-03) | SMBv1 — "The specified service does not exist" | Phase 1, Step 4 |
| [TS-04](#ts-04) | Snort path concatenation: `C:\Snort\etc\C:\Snort\rules/bad-traffic.rules` | Phase 2, Step 4 |
| [TS-05](#ts-05) | Snort ERROR: Unknown rule type: `dynamic` | Phase 2, Step 4 |
| [TS-06](#ts-06) | `Install-WindowsFeature` not recognized on Windows 10 | Phase 1, Step 4 |
| [TS-07](#ts-07) | Snort running but no alerts appearing in alert.ids | Phase 2, Step 7 |
| [TS-08](#ts-08) | Snort service install: "The system cannot find the file specified" | Phase 2, Step 7 |
| [TS-09](#ts-09) | Wazuh install: OS compatibility check failure | Phase 3, Step 1 |
| [TS-10](#ts-10) | Wazuh dashboard — where is the username and password? | Phase 3, Step 1 |
| [TS-11](#ts-11) | Wazuh MSI — "this installation package could not be opened" | Phase 3, Step 2 |
| [TS-12](#ts-12) | `NET START WazuhSvc` — "The service name is invalid" | Phase 3, Step 2 |
| [TS-13](#ts-13) | How to edit ossec.conf to add Snort log ingestion | Phase 3, Step 4 |

---

## TS-01

**Topic:** NIC 2 Bridged Adapter — Where Does the IP Come From?

**Section:** [Phase 1, Step 2 — Add NICs to CLIENT1](README.md#step-2--add-three-nics-to-client1)

**Question asked:** *"Also add a second NIC connected to your physical/bridged adapter — this is the internet-facing interface. What does this mean specifically?"*

**Explanation:**

The bridged adapter makes CLIENT1 appear as a separate physical device on your home LAN — getting its own IP address directly from your router, exactly like another laptop plugged into the network.

You have three options:

| Option | Method | Pros | Cons |
|---|---|---|---|
| A | Pure DHCP | Easy to start | IP can change, breaks port forwarding |
| B | DHCP reservation on router ✅ | Stable IP, no manual config inside VM | Requires router admin access |
| C | Fully static inside Windows | All config in one place | Must stay outside DHCP pool range |

**Recommended: Option B — DHCP Reservation**

1. Find CLIENT1's MAC address: `Get-NetAdapter` → note `MacAddress` for Ethernet 2
2. Log into your router admin page (usually `192.168.1.1` or `192.168.0.1`)
3. Find **DHCP Reservation** (also called "static DHCP" or "address reservation")
4. Bind CLIENT1's MAC to a fixed IP like `192.168.1.150`
5. Set router port-forwarding rules for ports 3389, 445, 21, 23 → `192.168.1.150`

CLIENT1 still gets its IP via DHCP but it will always be the same address.

---

## TS-02

**Topic:** Snort Dynamic Module Path Error (Linux paths on Windows)

**Section:** [Phase 2, Step 4 — Configure snort.conf](README.md#step-4--configure-snortconf)

**Error received:**
```
ERROR: C:\Snort\etc\snort.conf(249) Could not stat dynamic module path
"/usr/local/lib/snort_dynamicpreprocessor/": No such file or directory.
```

**Cause:** The default snort.conf ships with Linux-style paths that don't exist on Windows.

**Resolution:**

Open `C:\Snort\etc\snort.conf` as Administrator and use Ctrl+H to find and replace all Linux paths:

| Find | Replace with |
|---|---|
| `/usr/local/lib/snort_dynamicpreprocessor/` | `C:\Snort\lib\snort_dynamicpreprocessor` |
| `/usr/local/lib/snort_dynamicengine/` | `C:\Snort\lib\snort_dynamicengine` |
| `/usr/local/lib/snort_dynamicrules/` | `C:\Snort\lib\snort_dynamicrules` |

Then create the folders:
```cmd
mkdir C:\Snort\lib\snort_dynamicpreprocessor
mkdir C:\Snort\lib\snort_dynamicengine
mkdir C:\Snort\lib\snort_dynamicrules
```

Re-run the config test:
```cmd
C:\Snort\bin\snort.exe -T -i 2 -c C:\Snort\etc\snort.conf
```

---

## TS-03

**Topic:** SMBv1 — "The specified service does not exist"

**Section:** [Phase 1, Step 4 — Weaken CLIENT1](README.md#step-4--intentionally-weaken-client1)

**Error received:**
```
Set-SmbServerConfiguration : The specified service does not exist.
At line:1 char:1
+ Set-SmbServerConfiguration -EnableSMB1Protocol $true -Force
    + FullyQualifiedErrorId : Windows System Error 1243,Set-SmbServerConfiguration
```

**Cause:** On Windows 10, SMBv1 is disabled by default at the feature level. The PowerShell command can only configure the service — it can't enable the feature. The Windows feature must be installed first.

**Resolution:** Run in order:

```powershell
# Step 1 — Install the feature
Enable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol -All -NoRestart
Enable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol-Server -NoRestart

# Step 2 — Reboot (required)
Restart-Computer

# Step 3 — After reboot, start the service
Start-Service -Name LanmanServer
Set-Service -Name LanmanServer -StartupType Automatic

# Step 4 — Now configure SMBv1
Set-SmbServerConfiguration -EnableSMB1Protocol $true -Force

# Step 5 — Verify
Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol
Get-WindowsOptionalFeature -Online -FeatureName SMB1Protocol
```

If `Enable-WindowsOptionalFeature` fails, use DISM instead:
```cmd
dism /online /Enable-Feature /FeatureName:SMB1Protocol /All
```

---

## TS-04

**Topic:** Snort Path Concatenation Error

**Section:** [Phase 2, Step 4 — Configure snort.conf](README.md#step-4--configure-snortconf)

**Error received:**
```
ERROR: C:\Snort\etc\C:\Snort\rules/bad-traffic.rules(0) Unable to open rules file
"C:\Snort\etc\C:\Snort\rules/bad-traffic.rules": Invalid argument.
```

**Cause:** Two problems in snort.conf:

1. `WHITE_LIST_PATH` and `BLACK_LIST_PATH` are defined **twice** — the second definition (`../rules`) overrides the correct first definition and causes path concatenation
2. Rule include lines reference files you haven't downloaded (all the default rule files)

**Resolution:**

**Fix 1 — Remove the duplicate path definitions.** Search snort.conf for a second occurrence of these lines and comment them out:
```conf
# var WHITE_LIST_PATH ../rules    ← comment this out (second occurrence)
# var BLACK_LIST_PATH ../rules    ← comment this out (second occurrence)
```

**Fix 2 — Comment out all missing rule includes.** In Notepad, Ctrl+H:
- Find: `include $RULE_PATH/`
- Replace: `# include $RULE_PATH/`

Then manually remove the `#` from only these two lines:
```conf
include $RULE_PATH\community.rules
include $RULE_PATH\local.rules
```

Also comment out the SO rules section at the bottom:
```conf
# dynamic library rules
# include $SO_RULE_PATH/bad-traffic.rules
# include $SO_RULE_PATH/chat.rules
# ... (all commented out)
```

---

## TS-05

**Topic:** Snort "Unknown rule type: dynamic"

**Section:** [Phase 2, Step 4 — Configure snort.conf](README.md#step-4--configure-snortconf)

**Error received:**
```
ERROR: C:\Snort\etc\snort.conf(672) Unknown rule type: dynamic.
```

**Cause:** Line 672 contains the bare text `dynamic library rules` without a `#` comment character. Snort tries to parse `dynamic` as a rule type keyword and fails.

**Resolution:** Open snort.conf and find the line:
```
dynamic library rules
```
Add a `#` to make it a comment:
```conf
# dynamic library rules
```

Do the same for any other bare section headers nearby:
```conf
# decoder and preprocessor event rules
```

These are descriptive headings that were never meant to be parsed — they just need `#` prefixes.

---

## TS-06

**Topic:** `Install-WindowsFeature` Not Recognized on Windows 10

**Section:** [Phase 1, Step 4 — Weaken CLIENT1 (FTP)](README.md#step-4--intentionally-weaken-client1)

**Error received:**
```
Install-WindowsFeature : The term 'Install-WindowsFeature' is not recognized as the name
of a cmdlet, function, script file, or operable program.
```

**Cause:** `Install-WindowsFeature` is a Server Manager PowerShell cmdlet available only on **Windows Server**. CLIENT1 runs Windows 10, which uses DISM instead.

**Resolution:** Replace all `Install-WindowsFeature` commands with DISM equivalents:

```powershell
# Instead of: Install-WindowsFeature -Name Web-FTP-Server
dism /online /Enable-Feature /FeatureName:IIS-FTPServer /All

# Instead of: Install-WindowsFeature -Name Web-Server
dism /online /Enable-Feature /FeatureName:IIS-WebServer /All

# Instead of: Install-WindowsFeature -Name Web-Mgmt-Tools
dism /online /Enable-Feature /FeatureName:IIS-ManagementConsole /All
```

After DISM completes, configure FTP via **IIS Manager → Sites → Add FTP Site** (port 21, Anonymous auth, no SSL, path `C:\shares`).

---

## TS-07

**Topic:** Snort Running But No Alerts in alert.ids

**Section:** [Phase 2, Step 7 — Run Snort](README.md#step-7--run-snort)

**Symptom:** Pinged CLIENT1's bridged IP and received echo replies, but `C:\Snort\log\alert.ids` is empty or doesn't exist.

**Diagnosis checklist:**

**1 — Wrong interface index**
```cmd
C:\Snort\bin\snort.exe -W
```
Verify index 2 is actually your bridged adapter. Relaunch with the correct index.

**2 — ICMP rule threshold too high**

The original rule requires 5 pings in 5 seconds:
```snort
threshold:type threshold, track by_src, count 5, seconds 5
```
A default `ping` sends only 4 packets — not enough to trigger. Either:
- Send a continuous ping: `ping 192.168.1.150 -t`
- Or remove the threshold entirely from local.rules:
```snort
alert icmp any any -> $HOME_NET any (msg:"ICMP Ping Test"; sid:1000003; rev:1;)
```

**3 — HOME_NET subnet mismatch**

Check CLIENT1's actual bridged IP subnet:
```powershell
Get-NetIPAddress -InterfaceAlias "Ethernet 2" -AddressFamily IPv4
```
If your router uses `192.168.0.x` but snort.conf has `192.168.1.0/24`, no alerts fire. Match them.

**4 — Confirm Snort sees traffic at all**
```cmd
C:\Snort\bin\snort.exe -i 2 -v -c C:\Snort\etc\snort.conf -l C:\Snort\log -A full -K ascii
```
The `-v` flag shows packets scrolling on screen. If nothing scrolls while pinging, Snort is on the wrong interface.

---

## TS-08

**Topic:** Snort Windows Service Install Failure

**Section:** [Phase 2, Step 7 — Run Snort (Optional service)](README.md#step-7--run-snort)

**Error received:**
```
The system cannot find the file specified.
```

**Cause:** Windows can't locate `snort.exe` because the paths in the service install command aren't quoted, or the command is run from the wrong directory.

**Resolution — Option 1: Quoted full paths**
```cmd
"C:\Snort\bin\snort.exe" /SERVICE /INSTALL -i 2 -c "C:\Snort\etc\snort.conf" -l "C:\Snort\log" -A full -K ascii
```

**Resolution — Option 2: Register via SC**
```cmd
sc create SnortIDS binPath= "\"C:\Snort\bin\snort.exe\" -i 2 -c \"C:\Snort\etc\snort.conf\" -l \"C:\Snort\log\" -A full -K ascii" start= auto DisplayName= "Snort IDS"
sc start SnortIDS
```

**Resolution — Option 3: Background minimized window (simplest for lab)**

Skip the service entirely. Run Snort in a minimized window that stays open:
```cmd
start /min "Snort IDS" "C:\Snort\bin\snort.exe" -i 2 -c "C:\Snort\etc\snort.conf" -l "C:\Snort\log" -A full -K ascii
```
This is perfectly adequate for a portfolio lab environment.

---

## TS-09

**Topic:** Wazuh Install OS Compatibility Error

**Section:** [Phase 3, Step 1 — Deploy Wazuh Manager](README.md#step-1--deploy-wazuh-manager-vm)

**Error received:**
```
ERROR: The recommended systems are: Red Hat Enterprise Linux 7, 8, 9; CentOS 7, 8;
Amazon Linux 2; Ubuntu 16.04, 18.04, 20.04, 22.04. The current system does not match
this list. Use -i|--ignore-check to skip this check.
```

**Cause:** The Wazuh install script checks your Linux distribution before proceeding. If you're running a version not on the supported list (Ubuntu 23.x, 24.x, Debian, Mint, etc.) it refuses.

**Resolution:**

**Option 1 — Bypass the check (safe for labs):**
```bash
sudo bash wazuh-install.sh -a -i
```

**Option 2 — Verify your Ubuntu version:**
```bash
lsb_release -a
cat /etc/os-release
```
Wazuh 4.7.x officially supports Ubuntu 22.04 LTS. If you're on a different version and want full compatibility, rebuild the VM with Ubuntu 22.04 LTS from https://ubuntu.com/download/server.

---

## TS-10

**Topic:** Wazuh Dashboard — Username and Password

**Section:** [Phase 3, Step 1 — Deploy Wazuh Manager](README.md#step-1--deploy-wazuh-manager-vm)

**Question asked:** *"After install, note the auto-generated dashboard password. What is the username and password?"*

**Resolution:**

The credentials are auto-generated during install. Retrieve them on your Wazuh Manager Linux VM:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

If the tar file is no longer available:
```bash
sudo cat /var/log/wazuh-install.log | grep -i password
```

**Default username:** `admin`
**Password:** whatever was auto-generated (shown in the commands above)

**Accessing the dashboard:**
1. Open browser on your host machine
2. Navigate to `https://172.16.0.101` (use your Wazuh VM's actual IP)
3. You'll see a certificate warning — click **Advanced → Proceed anyway** (self-signed cert is normal for lab)
4. Login with `admin` + retrieved password

---

## TS-11

**Topic:** Wazuh Agent MSI — "This Installation Package Could Not Be Opened"

**Section:** [Phase 3, Step 2 — Install Wazuh Agent on CLIENT1](README.md#step-2--install-wazuh-agent-on-client1)

**Symptom:** Running `msiexec.exe /i wazuh-agent.msi ...` fails immediately with a package error.

**Cause:** The MSI file downloaded incompletely or was corrupted. A valid Wazuh agent MSI is approximately 50MB — anything smaller means the download failed partway through.

**Resolution:**

```powershell
# Step 1 — Check file size (should be ~50,000,000 bytes)
Get-Item "C:\Users\admin\Downloads\wazuh-agent.msi" | Select-Object Name, Length

# Step 2 — Delete and re-download using WebClient (shows completion)
Remove-Item "C:\Users\admin\Downloads\wazuh-agent.msi"

$client = New-Object System.Net.WebClient
$client.DownloadFile(
  "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.5-1.msi",
  "C:\Users\admin\Downloads\wazuh-agent.msi"
)
Write-Host "Download complete"

# Step 3 — Verify size again before installing
Get-Item "C:\Users\admin\Downloads\wazuh-agent.msi" | Select-Object Name, Length
```

> ⚠️ The agent version must match your Wazuh Manager version. If your manager is 4.7.5, download `wazuh-agent-4.7.5-1.msi`.

---

## TS-12

**Topic:** `NET START WazuhSvc` — "The service name is invalid"

**Section:** [Phase 3, Step 2 — Install Wazuh Agent on CLIENT1](README.md#step-2--install-wazuh-agent-on-client1)

**Error received:**
```
The service name is invalid.
```

**Cause:** The Wazuh agent service was never registered because the MSI installation failed silently (the `/q` quiet flag hides errors).

**Diagnosis:**

```powershell
# Check if any Wazuh service exists
Get-Service | Where-Object {$_.Name -like "*wazuh*" -or $_.Name -like "*ossec*"}

# Check if the agent folder was created
dir "C:\Program Files (x86)\ossec-agent"
```

If no service and no folder → the install failed.

**Resolution:**

Re-run the install with logging enabled (removes the silent `/q` flag):
```powershell
msiexec.exe /i "C:\Users\admin\Downloads\wazuh-agent.msi" `
  WAZUH_MANAGER="172.16.0.101" `
  WAZUH_AGENT_NAME="CLIENT1-Honeypot" `
  /l*v "C:\wazuh-install.log"
```

This opens the GUI installer and writes a log. Check for errors:
```powershell
Get-Content "C:\wazuh-install.log" | Select-String "error","fail" -CaseSensitive:$false
```

After successful install, try starting the service with alternate names (changed between versions):
```powershell
NET START WazuhSvc      # newer versions
NET START OssecSvc      # older versions
Start-Service -DisplayName "Wazuh"
```

If folder exists but service isn't registered:
```powershell
& "C:\Program Files (x86)\ossec-agent\wazuh-agent.exe" install-service
NET START WazuhSvc
```

---

## TS-13

**Topic:** How to Edit ossec.conf to Add Snort Log Ingestion

**Section:** [Phase 3, Step 4 — Ingest Snort Alerts](README.md#step-4--ingest-snort-alerts-into-wazuh)

**Question asked:** *"I'm not sure what I'm supposed to do to edit the ossec.conf file."*

**Full step-by-step:**

**Step 1 — Open Notepad as Administrator** (required for files in Program Files)

1. Click **Start**
2. Type `Notepad`
3. Right-click → **Run as Administrator** → Yes

**Step 2 — Open the file**

Inside Notepad:
- **File → Open**
- Navigate to: `C:\Program Files (x86)\ossec-agent\`
- Change file filter (bottom right) from `Text Documents (*.txt)` to `All Files (*.*)`
- Select `ossec.conf` → **Open**

**Step 3 — Find the insertion point**

Press **Ctrl+F** and search for:
```
</ossec_config>
```
This is the very last line of the file.

**Step 4 — Insert the Snort block**

Click your cursor just **before** `</ossec_config>` and add:

```xml
  <localfile>
    <log_format>snort-full</log_format>
    <location>C:\Snort\log\alert.ids</location>
  </localfile>

</ossec_config>
```

**Step 5 — Save**

**File → Save** (keep the same filename and location)

**Step 6 — Restart the agent**

```powershell
Restart-Service WazuhSvc

# Verify it picked up the Snort log source
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 30
```

Look for a line mentioning `snort` or `alert.ids` — this confirms Wazuh is now monitoring the Snort alert file.

---

*All issues documented above were encountered and resolved during the actual build of this lab.*

[← Back to Main Tutorial](README.md)
