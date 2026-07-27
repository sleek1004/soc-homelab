# Home SOC Lab: SIEM Detection Pipeline with Wazuh

A self-built Security Operations Centre lab running on Apple Silicon, demonstrating an end-to-end detection pipeline: log collection, SIEM analysis, endpoint telemetry, attack simulation, and alert triage mapped to MITRE ATT&CK.

Built as hands-on practice alongside my MSc in Cybersecurity Technology, and directly connected to my dissertation on reducing false-positive alert fatigue in Wazuh.

---

## What this project demonstrates

- Standing up and configuring a SIEM (Wazuh) from scratch
- Onboarding endpoints and forwarding security telemetry with a deployed agent
- Enriching Linux endpoint visibility with auditd
- Simulating a real attack (SSH brute force) and detecting it
- Reading and triaging alerts down to the raw log and field level
- Mapping detected activity to the MITRE ATT&CK framework

---

## Architecture

Four virtual machines on an isolated network, running under UTM on Apple Silicon (all native ARM64):

| Role | Machine | Purpose |
|------|---------|---------|
| SIEM server | Ubuntu 24.04 LTS | Wazuh manager, indexer, dashboard |
| Endpoint | Ubuntu 24.04 LTS | Monitored host with Wazuh agent + auditd |
| Attacker | Kali Linux | Attack simulation (Hydra) |

Data flow: the attacker targets the endpoint, the endpoint's telemetry is forwarded by the Wazuh agent to the SIEM server, which analyses it and surfaces alerts in the dashboard.

```
Kali (attacker)  ->  Ubuntu endpoint (agent + auditd)  ->  Wazuh server (SIEM)  ->  Dashboard
```

---

## Build summary

### 1. SIEM server
- Deployed Ubuntu Server 24.04 LTS as a native ARM64 VM (chosen over emulated x86 for stability and performance)
- Installed the full Wazuh stack (manager, indexer, dashboard) via the assisted installer
- Secured dashboard access and managed admin credentials

### 2. Endpoint onboarding
- Deployed the Wazuh agent on the monitored Ubuntu host and enrolled it against the manager
- Verified the agent registered and reported as Active in the dashboard
- Resolved a broken package state during agent install by clearing stale dpkg metadata (documented in the troubleshooting notes)

### 3. Telemetry enrichment
- Installed and enabled auditd for kernel-level audit logging
- Added a watch rule on `/etc/passwd` to capture identity-related changes
- Confirmed events were captured locally and forwarded to the SIEM

### 4. Attack simulation
- Ran a controlled SSH brute-force attack from Kali using Hydra against the endpoint
- Kept the attack scoped (small wordlist, limited parallelism) to trigger detection without disruption

### 5. Detection and triage
- Wazuh detected the repeated authentication failures and raised a brute-force alert
- The activity was automatically mapped to MITRE ATT&CK technique T1110 (Brute Force / Password Guessing)
- Triaged the alert down to source IP, targeted account, rule level, and the raw log line

---

## Detection result

The SSH brute force generated a burst of failed authentication events from a single source. Wazuh's correlation rules identified the pattern and raised an alert.

Key fields from a single detected event:

| Field | Value | Meaning |
|-------|-------|---------|
| rule.description | sshd: authentication failed | What fired |
| data.srcip | (attacker IP) | Source of the attack |
| data.dstuser | (target account) | Account targeted |
| agent.name | ubuntu-endpoint | Host attacked |
| rule.firedtimes | 5+ | Repetition = brute-force signature |
| MITRE ATT&CK | T1110 | Brute Force |

---

## Skills evidenced

- SIEM deployment and administration (Wazuh)
- Endpoint detection and telemetry (agent management, auditd)
- Log analysis and alert triage
- Attack simulation and validation
- MITRE ATT&CK mapping
- Linux system administration and troubleshooting
- Virtualisation and lab design

---

## Frameworks referenced

- MITRE ATT&CK (technique mapping for detected activity)
- NIST CSF (Detect and Respond functions)
- CIS Benchmarks (endpoint configuration assessment via Wazuh SCA)

---

## Next steps

- Detection engineering: writing and tuning custom detection rules, measured with precision, recall, and false-positive rate (the focus of my MSc dissertation on reducing SIEM alert fatigue)
- Adding Sysmon for Linux to align telemetry with the Windows Sysmon schema
- Expanding attack coverage with additional MITRE ATT&CK techniques

---

## Repository structure

```
/docs           Detailed build and troubleshooting notes
/screenshots    Dashboard and detection evidence (credentials redacted)
README.md       This overview
```

*All work performed in an isolated lab environment against systems I own. Credentials and secrets have been redacted from all documentation and screenshots.*
