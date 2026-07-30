# Wazuh-SIEM-lab-detecting-a-simulated-brute-force-attack-T1110
SOC detection lab: deployed Wazuh SIEM, simulated a brute-force attack (MITRE T1110) against a Windows endpoint, and produced a full incident report from the resulting alerts, covering log correlation, triage, and analyst write-up.
# Wazuh SIEM Brute-Force Detection Lab

A hands-on detection engineering project: deploy Wazuh SIEM, onboard a Windows endpoint, simulate a network-based brute-force credential attack, and produce a full SOC incident report from the resulting alerts.

**Status:** Complete ✅ | **Severity:** Medium | **MITRE ATT&CK:** [T1110 – Brute Force](https://attack.mitre.org/techniques/T1110/)

---

## 📋 Project Summary

This project simulates and detects a brute-force authentication attack against a local Windows account, using a self-hosted Wazuh SIEM deployment. The goal was to build practical, end-to-end SOC analyst skills: environment setup, log source configuration, attack simulation, alert triage, and formal incident reporting — the same workflow used in a real security operations center.

**What this project demonstrates:**
- Standing up and configuring a Wazuh SIEM manager + agent architecture
- Enabling and tuning Windows Security auditing for authentication events
- Simulating an attacker technique (T1110 – Brute Force) in a safe, isolated lab
- Reading and interpreting raw Windows Event Log data (Event ID 4625)
- Understanding Wazuh's rule correlation engine (single-event vs. aggregated detection)
- Mapping detections to the MITRE ATT&CK framework
- Writing a professional, stakeholder-ready incident report

---

## 🏗️ Lab Environment

| Component | Details |
|---|---|
| SIEM | Wazuh (manager + indexer + dashboard) |
| Endpoint | Windows 10, Wazuh agent installed |
| Network | Isolated home-lab VM network (no production systems) |
| Log Source | Windows Security Event Log (`Microsoft-Windows-Security-Auditing`) |
| Simulation Tool | PowerShell (scripted authentication loop over SMB/NTLM) |

**Architecture:**
```
[Windows 10 Endpoint] --(Wazuh Agent)--> [Wazuh Manager] --> [Wazuh Indexer] --> [Wazuh Dashboard]
        │
        └─ Security Event Log (Event ID 4625) forwarded via eventchannel
```

---

## 🎯 Attack Simulation

Windows logon auditing (`auditpol /set /subcategory:"Logon" /success:enable /failure:enable`) was enabled to ensure failed logon events (Event ID 4625) were captured. A PowerShell script then generated repeated failed authentication attempts against a local test account (`testuser`) over the network authentication path (Logon Type 3, NTLM) — simulating how brute-force tooling authenticates programmatically rather than via the interactive console.

```powershell
$target = "testuser"
$wrongPasswords = "Passw0rd1","Passw0rd2","Passw0rd3","Passw0rd4","Passw0rd5","Passw0rd6","Passw0rd7","Passw0rd8"

foreach ($p in $wrongPasswords) {
    $securePwd = ConvertTo-SecureString $p -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential("$env:COMPUTERNAME\$target", $securePwd)
    try {
        New-PSDrive -Name "TempTest" -PSProvider FileSystem -Root "\\$env:COMPUTERNAME\C$" -Credential $cred -ErrorAction Stop
    } catch {
        Write-Host "Failed attempt with password: $p"
    }
    Start-Sleep -Milliseconds 800
}
```

---

## 🔍 Detection

Eight failed logon attempts were logged individually by Wazuh (**Rule 60122**, level 5 — "Logon Failure – Unknown user or bad password"). Wazuh's built-in correlation engine tracked the frequency of these events and, once the threshold of 8 matching events was crossed within its detection window, escalated to a single high-severity alert:

| | |
|---|---|
| **Rule ID** | 60204 |
| **Description** | Multiple Windows Logon Failures |
| **Severity Level** | 10 (escalated from level 5) |
| **MITRE ATT&CK** | T1110 – Brute Force (Credential Access) |
| **Trigger** | 8 failed logons within the correlation window |

This is the core detection engineering concept the project demonstrates: **individually low-signal events becoming a high-confidence alert through correlation**, exactly how production SIEMs cut down alert fatigue while still catching real attack patterns.

### Timeline

| Time (UTC) | Event |
|---|---|
| 15:45:21 | PowerShell script execution detected (Rule 91816) |
| 15:45:25 – 15:45:58 | 8 failed logon attempts (Rule 60122 × 8) |
| 15:47:17 | Correlation alert fires — Rule 60204, Level 10 |

---

## 🧠 Analysis Highlights

- **Automation indicator:** Attempts landed at near-identical ~3-second intervals — irregular in human typing, but consistent with scripted execution.
- **Process correlation:** A PowerShell execution event immediately preceded the first failed logon, tying the authentication failures to an identifiable process rather than treating them as an isolated anomaly.
- **Accurate scoping:** The source address was an IPv6 link-local address resolving to the same host — the report explicitly notes this as a *self-originated, network-vector simulation* rather than overclaiming an external attacker, since precision matters more than a dramatic narrative.
- **Detection latency observation:** ~79 seconds elapsed between the final failed logon and the correlation alert firing — flagged as a tuning opportunity in the report's Lessons Learned section.

---

## 📄 Full Incident Report

The complete, formatted incident report — covering executive summary, timeline, affected user/host, detection details, impact assessment, and remediation recommendations — is available here:

📎 [`SOC_Incident_Report_BruteForce.pdf`](SOC_Incident_Report_BruteForce.pdf) *(view in-browser)*  


---

## 🖼️ Screenshots

| Description | File |
|---|---|
| Individual failed logon alert (Rule 60122) |<img width="1908" height="938" alt="image" src="https://github.com/user-attachments/assets/c3809a2a-3941-4e9c-9894-035029daeec0" />
 |
| Escalated correlation alert (Rule 60204) | <img width="1916" height="960" alt="Screenshot 2026-07-30 164030" src="https://github.com/user-attachments/assets/43bd03cd-61dc-4e42-8d0b-4dcdf2c14b62" />
 |
| Wazuh Discover view — escalation sequence |<img width="1889" height="482" alt="Screenshot 2026-07-30 164800" src="https://github.com/user-attachments/assets/2c7a5547-eac6-499a-807d-71ac1fab9aff" />
  |
| Raw JSON of correlation alert | <img width="826" height="752" alt="image" src="https://github.com/user-attachments/assets/fa819383-6075-4b96-9a0d-de8ded0b28de" />
 |


## 🛠️ Skills Demonstrated

- SIEM deployment and configuration (Wazuh manager/agent)
- Windows Security auditing and Event Log analysis
- Attack simulation and adversary technique emulation (MITRE ATT&CK)
- Alert triage and log correlation
- Incident documentation and report writing
- Analytical reasoning: distinguishing benign vs. malicious behavior patterns

---


---

## 🔗 Next Steps / Future Iterations

- Simulate the same technique from a second VM with a distinct routable IP to model a genuinely external attacker
- Add a custom Wazuh rule to lower detection latency for this technique
- Extend the lab with Sysmon for process-to-network correlation on future scenarios (e.g. PowerShell obfuscation, LOLBin abuse)

---

*This project was built as a self-directed learning exercise to develop practical SOC analyst and detection engineering skills.*
