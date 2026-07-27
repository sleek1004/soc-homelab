# Screenshots

Evidence from the home SOC lab, showing the SSH brute-force attack detected end to end. All captures are from an isolated lab environment against a host I own. No real credentials are shown; lab IPs (192.168.64.x) are private addresses.

### 01-detection-dashboard.png
The Wazuh Threat Hunting dashboard filtered to `sshd` over the last 24 hours. Shows 10 authentication failures and 0 successes from the brute-force attempt, with the activity classified in the MITRE ATT&CK panel as Password Guessing and SSH.

### 02-alert-details.png
A single detected event expanded to its full field set. Key fields: `data.srcip` (the attacker, Kali), `data.dstuser` (the targeted account), `rule.firedtimes` (repetition count, the brute-force signature), and `full_log` (the raw log line the alert was decoded from).

### 03-hydra-attack.png
The attack itself, run from Kali using Hydra against the endpoint over SSH. Shows the login attempts and the run completing. The failed attempts are what the SIEM detected.

### 04-agent-active.png
The Wazuh agent running on the monitored Ubuntu endpoint, with all core processes active. Confirms the endpoint is onboarded and forwarding telemetry to the SIEM.
