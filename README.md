# SOC Home Lab: Splunk SIEM, Sysmon & Atomic Red Team

A self-hosted, isolated Security Operations Center (SOC) home lab built on VMware Workstation to practice log ingestion, endpoint telemetry, attack simulation (MITRE ATT&CK), and detection engineering using Splunk SPL.

---

## 1. Lab Architecture & Design
To optimize host resources (16 GB RAM total), Phase 1 consolidates the SIEM and endpoint telemetry on a single Windows Server 2022 VM (`SOC-LAB-01`), while the Windows 11 host acts as the SOC Analyst workstation accessing Splunk Web remotely.

![VMware Hardware Configuration](screenshots/SOC_Lab_Architecture.png)

---

## 2. Step 1: VM Provisioning & Windows Server 2022 Setup
* **Hypervisor:** VMware Workstation
* **VM Name:** `SOC-LAB-01`
* **Resources:** 6144 MB RAM, 2 vCPUs, 100 GB Dynamically Allocated Disk, UEFI Firmware.
* **OS Edition:** Selected **Windows Server 2022 Standard Evaluation (Desktop Experience)** rather than Server Core to allow GUI-based troubleshooting, local browser verification, and easier management during initial lab setup.
* **Configuration:** Left as a Standalone Workgroup (no Active Directory or extra Server Roles installed in Phase 1).
* **Initial Network:** Set to **NAT (`VMnet8`)** temporarily to download tools and configurations.

<!-- Screenshot 1: VMware VM Settings showing RAM, CPU, Disk, and OS -->
![VMware Hardware Configuration](screenshots/01_vm_settings.png)

---

## 3. Step 2: Splunk Enterprise Installation & Index Configuration
Installed **Splunk Enterprise (Windows 64-bit)** in `C:\Program Files\Splunk` running as `Local System` with Web Port `8000` and Management Port `8089`.

### Issue Encountered: Blank "Manage Indexes" Page
* **Problem:** Accessing `http://localhost:8000/en-US/manager/launcher/data/indexes` inside Windows Server rendered a blank white page below the top navigation bar because the legacy Microsoft Edge version bundled with Windows Server 2022 failed to render Splunk's modern JavaScript UI.
* **Solution:** Allowed inbound `TCP 8000` through Windows Defender Firewall and accessed Splunk Web directly from the updated browser on the Windows 11 host.

```cmd
:: Check VM IP address
ipconfig

:: Allow Splunk Web port 8000 in Windows Firewall
netsh advfirewall firewall add rule name="Splunk Web" dir=in action=allow protocol=TCP localport=8000
```

<!-- Screenshot 2: Splunk Manage Indexes showing wineventlog and sysmon enabled -->
![Splunk Custom Indexes](screenshots/02_splunk_indexes.png)

### Data Collection Setup
1. Created two dedicated indexes under **Settings → Indexes**:
   * `wineventlog` (for native Windows Security & System logs)
   * `sysmon` (for Sysmon operational telemetry)
2. Enabled **Local Event Log Collection** (**Settings → Data inputs**) for `Security` and `System` logs, routed to `index=wineventlog`.
3. Verified ingestion in **Search & Reporting** using `index=wineventlog`.

---

## 4. Step 3: Sysmon Deployment & Splunk Integration
Deployed **Microsoft Sysmon** with the **SwiftOnSecurity** configuration to capture high-fidelity endpoint telemetry (process creation, command lines, network connections, and registry modifications).

### Issue Encountered: PowerShell Cmdlets in CMD
* **Problem:** Running `Invoke-WebRequest` directly in Command Prompt returned `'Invoke-WebRequest' is not recognized`.
* **Solution:** Switched the terminal session to PowerShell (`powershell`) before executing download and archive commands.

### Commands Executed
```powershell
# Switch to PowerShell and create directory
mkdir C:\Sysmon
cd C:\Sysmon

# Download Sysmon and SwiftOnSecurity configuration
Invoke-WebRequest -Uri "[https://download.sysinternals.com/files/Sysmon.zip](https://download.sysinternals.com/files/Sysmon.zip)" -OutFile "Sysmon.zip"
Expand-Archive -Path "Sysmon.zip" -DestinationPath "C:\Sysmon" -Force
Invoke-WebRequest -Uri "[https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml](https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml)" -OutFile "sysmonconfig-export.xml"

# Install Sysmon with the config file
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

<!-- Screenshot 3: PowerShell output showing Sysmon64 installed and started -->
![Sysmon Installation Output](screenshots/03_sysmon_install.png)

### Configuring Splunk to Ingest Sysmon Logs
Appended the Sysmon operational log stanza to `inputs.conf` with XML rendering enabled for clean field extraction:

```powershell
Add-Content -Path "C:\Program Files\Splunk\etc\system\local\inputs.conf" -Value "`n[WinEventLog://Microsoft-Windows-Sysmon/Operational]`ndisabled = false`nindex = sysmon`nrenderXml = true"
```

### Issue Encountered: `Splunkd` Service Restart Failure
* **Problem:** Running `Restart-Service Splunkd` in PowerShell failed with `Cannot stop Splunkd service on computer '.'`.
* **Solution:** Restarted Splunk cleanly using its native CLI executable:

```powershell
& "C:\Program Files\Splunk\bin\splunk.exe" restart
```

Verified log ingestion in Splunk using `index=sysmon` (500+ events received immediately).

<!-- Screenshot 4: Splunk Search & Reporting showing Sysmon events arriving in index=sysmon -->
![Sysmon Events in Splunk](screenshots/04_sysmon_events_splunk.png)

---

## 5. Step 4: Atomic Red Team Installation
Installed **Invoke-AtomicRedTeam** and the Atomics library to simulate MITRE ATT&CK techniques safely inside the lab.

### Issue Encountered: Windows Defender / AMSI Block
* **Problem:** Running `IEX (IWR ...)` triggered `ScriptContainedMaliciousContent` as Windows Defender blocked the in-memory installation script before it reached the excluded directory.
* **Solution:** Added a folder exclusion for `C:\AtomicRedTeam`, temporarily disabled Real-Time Monitoring inside the isolated lab VM during installation, and verified the framework.

```powershell
# Exclude Atomic Red Team directory and temporarily disable Real-Time Monitoring for installation
Add-MpPreference -ExclusionPath "C:\AtomicRedTeam"
Set-MpPreference -DisableRealtimeMonitoring $true

# Install Invoke-AtomicRedTeam and Atomics library
Set-ExecutionPolicy Bypass -Scope Process -Force
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
IEX (IWR '[https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1](https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1)' -UseBasicParsing)
Install-AtomicRedTeam -getAtomics -Force

# Verify installation by listing tests for T1059.001 (PowerShell)
Invoke-AtomicTest T1059.001 -ShowDetailsBrief
```

<!-- Screenshot 5: PowerShell output listing Atomic Red Team tests for T1059.001 -->
![Atomic Red Team Verification](screenshots/05_atomic_red_team_verify.png)

---

## 6. Step 5: Network Isolation (Host-Only) & Troubleshooting
Before running any attack simulations, the VM was isolated from the internet and home network by switching its VMware Network Adapter from **NAT** to **Host-Only (`VMnet1`)**.

### Issue Encountered: Connection Timeout on Host-Only Network
* **Problem:** After switching to `Host-Only`, the VM received IP `192.168.65.130` (with no Default Gateway), but accessing `http://192.168.65.130:8000` from Windows 11 timed out.
* **Root Cause:**
  1. The firewall rule needed to apply across all network profiles (`Public`/`Private`/`Domain`) since Host-Only is classified as an Unidentified/Public network.
  2. Running `ipconfig` on the Windows 11 host revealed that `VMware Network Adapter VMnet1` had a mismatched static IP (`192.168.10.2/24` with gateway `192.168.10.1`) on a different subnet than the VM (`192.168.65.x`).
* **Solution:**
  1. Updated the Windows Server firewall rule to apply to all profiles:
     ```powershell
     netsh advfirewall firewall set rule name="Splunk Web" new profile=any
     ```
  2. Reconfigured `VMnet1` IPv4 settings on Windows 11 (`ncpa.cpl`) to match the VM subnet:
     * **IP Address:** `192.168.65.1`
     * **Subnet Mask:** `255.255.255.0`
     * **Default Gateway:** *(Left empty)*
  3. Verified full Splunk Web connectivity at `http://192.168.65.130:8000` while maintaining complete network isolation.
  4. Took a clean VMware Snapshot (`Clean-Lab-Ready`) as a safe rollback point before attack execution.

<!-- Screenshot 6: VM ipconfig showing Host-Only IP with empty Default Gateway -->
![Network Isolation Proof](screenshots/06_network_isolation.png)

---

---

## 7. Phase 2: Attack Simulations & Detection Engineering

### Detection 1: System Information Discovery (`T1082`)
* **MITRE ATT&CK Tactic:** Discovery (`T1082`)
* **Data Source:** Sysmon (`Event ID 1` - Process Creation)
* **Simulation Command:**
  ```powershell
  Invoke-AtomicTest T1082 -TestNumbers 1
  ```
* **Splunk SPL Query:** [View Query](detections/T1082_system_info.spl)
* **Analyst Findings (Process Tree):**
  * The query returned **2 events**, revealing the full execution chain: `powershell.exe` → `cmd.exe` → `systeminfo.exe`.
  * Identifying `powershell.exe` spawning `cmd.exe` to run reconnaissance commands is a strong indicator of scripted/automated discovery rather than normal administrative usage.
* **Troubleshooting Note (XML Parsing):**
  * Initial queries using `EventCode=1` returned no results because `renderXml = true` ingested raw XML without the Sysmon TA installed.
  * Resolved temporarily using SPL `rex` field extraction, and permanently by switching `renderXml = false` in `inputs.conf`.

![T1082 Detection in Splunk](screenshots/07_t1082_detection.png)

### Detection 2: Encoded & Hidden PowerShell Execution (`T1059.001`)
* **MITRE ATT&CK Tactic:** Execution (`T1059.001`)
* **Data Source:** Sysmon (`Event ID 1` - Process Creation)
* **Alert Severity:** High
* **Simulation Command (Offline Base64 Execution):**
  ```powershell
  $cmd = "whoami; hostname; Get-Date"
  $encoded = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))
  powershell.exe -NoProfile -WindowStyle Hidden -EncodedCommand $encoded
  ```
* **Splunk SPL Query:** [View Query](detections/T1059_001_encoded_powershell.spl)
* **Analyst Triage & Findings:**
  * Detected `powershell.exe` launching with suspicious evasion flags (`-NoProfile`, `-WindowStyle Hidden`, `-EncodedCommand`).
  * Extracted the Base64 payload from the `CommandLine` field and decoded it (UTF-16LE) to reveal the underlying reconnaissance commands (`whoami; hostname; Get-Date`).

![T1059.001 Detection in Splunk](screenshots/08_t1059_001_detection.png)
![Decoded Base64 Payload](screenshots/09_t1059_001_decoded.png)

### Detection 3: Scheduled Task Persistence (`T1053.005`)
* **MITRE ATT&CK Tactic:** Persistence (`T1053.005`)
* **Data Source:** Sysmon (`Event ID 1` - Process Creation)
* **Alert Severity:** High
* **Simulation & Cleanup Commands:**
  ```powershell
  Invoke-AtomicTest T1053.005 -TestNumbers 1
  Invoke-AtomicTest T1053.005 -TestNumbers 1 -Cleanup
  ```

  ---

## 8. SOC Monitoring Dashboard
To centralize visibility across the lab, I built a Splunk Dashboard that tracks Sysmon ingestion health, Event ID distribution, process creation timelines, and a unified **MITRE ATT&CK Detection Table** mapping all simulated techniques dynamically using SPL `eval/case`.

![SOC Threat Detection Dashboard](screenshots/11_soc_dashboard.png)
* **Splunk SPL Query:** [View Query](detections/T1053_005_scheduled_task.spl)
* **Analyst Triage & Findings:**
  * Detected `cmd.exe` spawning `schtasks.exe /create` to establish two persistence mechanisms: one triggered on user logon (`/sc onlogon`) and a high-privilege task running as `SYSTEM` on startup (`/sc onstart /ru system`).

![T1053.005 Detection in Splunk](screenshots/10_t1053_005_detection.png)
