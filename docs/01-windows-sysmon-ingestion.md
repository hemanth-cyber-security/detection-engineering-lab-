# Windows Sysmon Telemetry Ingestion with Wazuh

**Project:** Detection Engineering Lab  
**Milestone:** 01 — Windows Endpoint Telemetry Ingestion  
**Status:** Completed — basic telemetry collection and archive search verified  
**Environment:** Local virtualized lab  
**Cost:** Free and open-source components

---

## 1. Objective

The objective of this project was to build a local Windows endpoint telemetry pipeline using Sysmon and Wazuh.

The lab was designed to collect Windows process-creation events, forward them to a central Wazuh manager, index raw events, and make them searchable for future detection engineering activities.

This milestone establishes the telemetry foundation required to develop, test, and validate custom security detection rules.

### Learning objectives

- Deploy and configure a Wazuh all-in-one server.
- Enroll a Windows endpoint with the Wazuh agent.
- Install Sysmon and generate process-creation telemetry.
- Configure Wazuh to collect the Sysmon Operational event channel.
- Enable raw JSON event archiving and archive indexing.
- Troubleshoot the difference between security alerts and raw event archives.
- Verify that endpoint telemetry is searchable in Wazuh Discover.

---

## 2. Lab Architecture

The lab uses two virtual machines and the host computer.

| Component | Role | Status |
|---|---|---|
| Ubuntu Server 24.04 | Wazuh manager, indexer, and dashboard | Installed |
| Windows 11 VM | Monitored endpoint | Connected |
| Sysmon | Windows process and system activity telemetry | Installed |
| Wazuh agent | Collects and forwards Windows events | Active |
| Filebeat | Forwards Wazuh alert and archive data to the indexer | Archive indexing enabled |
| Wazuh Discover | Searches indexed events | Calculator and Notepad events visible |

### Data flow

```text
Windows 11 Endpoint
        |
        v
Sysmon Operational Event Log
        |
        v
Wazuh Windows Agent
        |
        v
Wazuh Manager
        |
        +----> Raw JSON Event Archive
        |
        v
Filebeat
        |
        v
Wazuh Indexer
        |
        v
Wazuh Discover
```

**Note:** This diagram represents the configured collection and indexing workflow. It does not imply that every event generates a Wazuh security alert.

---

## 3. Environment and Setup

### 3.1 Wazuh server

An Ubuntu Server 24.04 virtual machine was configured with the Wazuh all-in-one deployment.

The deployment includes the manager, indexer, and dashboard.

**Verified:** The dashboard was accessible, the Windows agent connected successfully, and the archive index was subsequently used to search raw Sysmon events.

### 3.2 Windows endpoint enrollment

The Wazuh agent was installed on the Windows 11 virtual machine and configured to communicate with the Wazuh manager.

The agent appeared in the Wazuh interface as:

- **Agent name:** `DE-WIN11-01`
- **Agent ID:** `001`
- **Status:** `Active`

TCP connectivity to port `1514` was also tested successfully.

### 3.3 Sysmon installation

Sysmon version 15.22 was installed on the Windows endpoint.

The Sysmon service was verified as running, and the following event channel was available:

```text
Microsoft-Windows-Sysmon/Operational
```

Local verification confirmed that Sysmon was generating process-creation events.

---

## 4. Wazuh Agent Configuration

The Windows agent was configured to collect the Sysmon Operational event channel.

The relevant configuration block in `ossec.conf` is:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

This block must be placed inside the existing `<ossec_config>` section.

**Configuration location used:**

```text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

The Wazuh agent service was restarted after the configuration was updated.

### Configuration validation

The following checks were performed during troubleshooting:

1. Confirmed that the Sysmon service was running.
2. Confirmed that the Sysmon Operational channel existed.
3. Queried local Sysmon events using PowerShell.
4. Confirmed that Sysmon process-creation events were present locally.
5. Confirmed that events were arriving in the Wazuh manager's raw archive.

**Result:** Basic Windows-to-Wazuh telemetry collection was verified.

---

## 5. Sysmon Event ID 1 — Process Creation

Sysmon Event ID 1 records process-creation activity. Its fields can support investigations into process execution, parent-child relationships, command-line activity, and user context.

### Important fields

| Field | Purpose |
|---|---|
| `Image` | Executable path of the created process |
| `CommandLine` | Command used to launch the process |
| `ParentImage` | Executable path of the parent process |
| `ProcessId` | Process identifier |
| `ParentProcessId` | Parent process identifier |
| `User` | Security context associated with the event |
| `Hashes` | File hash information, when provided |

### Local validation

Controlled test activity was generated on the Windows VM by launching Calculator and Notepad.

The Sysmon Operational event log was queried through PowerShell, and process-creation events were observed.

Example commands used to generate test activity:

```powershell
Start-Process notepad.exe
Start-Process calc.exe
```

These commands generate test activity; they do not themselves confirm that an event was collected. Verification was performed separately by examining Sysmon events.

### Example local event query

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Microsoft-Windows-Sysmon/Operational'
    Id = 1
    StartTime = (Get-Date).AddMinutes(-15)
} -MaxEvents 20 |
Select-Object TimeCreated, Id, Message |
Format-List
```

**Verified:** Local process-creation events were visible in PowerShell, including process activity associated with the lab's test applications.

**Not yet verified:** A complete, repeatable test procedure that correlates each generated process with its corresponding indexed event and records its event timestamp.

---

## 6. Raw Event Archiving and Archive Indexing

During troubleshooting, process-creation events were present in the Wazuh manager's raw JSON archive but were not initially visible in the dashboard's alert index.

### 6.1 Enable raw JSON archiving

The following setting was enabled in:

```text
/var/ossec/etc/ossec.conf
```

Configuration:

```xml
<logall_json>yes</logall_json>
```

This setting enables the Wazuh manager to archive received events in JSON format.

### 6.2 Enable Filebeat archive indexing

The Filebeat configuration was updated in:

```text
/etc/filebeat/filebeat.yml
```

The relevant settings were:

```yaml
  alerts:
    enabled: true
  archives:
    enabled: true
```

The configuration was validated, and Filebeat was restarted.

Commands used:

```bash
sudo filebeat test config
sudo systemctl restart filebeat
sudo systemctl status filebeat --no-pager
```

### 6.3 Validate archive ingestion

The raw archive was inspected for process names associated with the test activity.

Example command:

```bash
sudo grep -iE \
'notepad\.exe|calc\.exe|CalculatorApp\.exe' \
/var/ossec/logs/archives/archives.json | tail -10
```

**Verified:** Matching events were present in the archive file, and Calculator and Notepad events subsequently became searchable in Wazuh Discover after archive indexing was enabled.

---

## 7. Troubleshooting: Alerts vs. Archives

One of the key lessons from this milestone was the distinction between raw event collection and security alert generation.

### The issue

The dashboard initially used the `wazuh-alerts-*` data view.

Although Sysmon was generating process-creation events locally and the manager was receiving them, the test Calculator and Notepad events were not visible in the alert search.

### Root cause

Raw-event archiving and archive indexing were not initially configured end to end.

The investigation established that:

1. Sysmon was generating local events.
2. The Wazuh agent was connected.
3. Matching events were present in the manager's JSON archive.
4. Archive indexing was disabled in Filebeat.
5. After enabling archive indexing, the test events became searchable in Discover.

### Alerts versus archives

| Data source | Purpose |
|---|---|
| `wazuh-alerts-*` | Events that generated Wazuh alerts |
| `wazuh-archives-*` | Raw events received and archived by Wazuh, when archive indexing is configured |

The exact index names and available fields should be confirmed against the deployed Wazuh version.

**Important lesson:** The absence of an event from the alerts index does not prove that the agent failed to collect it. An event may be archived without matching a rule that produces a security alert.

---

## 8. Validation Results

| Test | Result | Evidence |
|---|---|---|
| Wazuh dashboard accessible | Pass | Dashboard accessed successfully |
| Windows agent enrollment | Pass | `DE-WIN11-01`, agent ID `001`, Active |
| TCP connectivity to port 1514 | Pass | Connection test succeeded |
| Sysmon service running | Pass | Service status checked |
| Sysmon Operational channel available | Pass | Channel enumerated |
| Local process-creation events | Pass | Events visible through PowerShell |
| Wazuh manager receiving raw events | Pass | Matching events found in `archives.json` |
| Archive indexing enabled | Pass | Filebeat archive setting changed and service restarted |
| Calculator and Notepad searchable in Discover | Pass | User confirmed the events were visible |
| Custom detection rule generates an alert | Not tested | Future milestone |
| Detection false-positive testing | Not tested | Future milestone |
| End-to-end test evidence captured and committed to GitHub | Not verified | Documentation task |

---

## 9. Limitations and Considerations

### Raw-event storage

Enabling raw JSON archiving and archive indexing increases storage consumption.

For this lab, archive logging is enabled to support investigation and detection development. Disk usage and retention should be monitored as more telemetry sources are added.

### Event coverage

This milestone focuses on Sysmon process-creation telemetry. It does not establish that all Sysmon event types are being collected or that every relevant process activity is detected.

### Detection coverage

No custom detection rule has been implemented or validated as part of this milestone.

Successful telemetry ingestion is a prerequisite for detection engineering, not evidence of complete detection coverage.

### Security and evidence handling

The lab should not contain real credentials, authentication keys, or unnecessary sensitive production data. Screenshots and exported logs should be reviewed and sanitized before publication.

### Reproducibility

The configuration snippets are documented, but a fresh rebuild from a clean environment has not yet been tested. Exact package versions, deployment commands, and evidence should be captured as the project develops.

---

## 10. Next Steps

The next phase focuses on turning collected telemetry into actionable detections.

### Milestone 02 — First Detection Rule

- Explore Sysmon Event ID 1 fields in Wazuh Discover.
- Establish a specific detection hypothesis.
- Implement a custom Wazuh rule for a controlled test behavior.
- Generate the behavior in the isolated Windows lab.
- Confirm that the expected alert is generated.
- Test benign activity to identify false positives.
- Document the detection logic, expected fields, and limitations.
- Map the behavior to MITRE ATT&CK where appropriate.
- Store the rule and repeatable test instructions in GitHub.

### Later milestones

- Expand Windows telemetry coverage.
- Add web server and network telemetry.
- Develop additional detections and tuning procedures.
- Introduce detection tests and version-controlled rule changes.
- Explore local AI assistance for rule drafting and test-case generation.
- Add cloud telemetry when the foundational detections are stable.

---

## 11. Conclusion

This milestone established a working local telemetry pipeline between a Windows 11 endpoint and Wazuh.

Sysmon process-creation events were verified locally, received by the Wazuh manager, archived in JSON, indexed, and made searchable in the dashboard.

The troubleshooting process also demonstrated why analysts must distinguish between raw event ingestion and alert generation when validating SIEM data pipelines.

The next milestone is to build and validate the first custom detection rule using the telemetry collected by this lab.

---

**Project status:** Windows Sysmon telemetry ingestion and archive search verified. Custom detection engineering remains the next milestone.
