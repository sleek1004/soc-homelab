# Build and Troubleshooting Notes

Detailed record of building the home SOC lab, including the problems hit along the way and how they were resolved. The troubleshooting is included deliberately: diagnosing and recovering from failures is the core of operational security work, and every issue below reflects something that happens on real systems.

Environment: UTM on Apple Silicon (macOS). All VMs native ARM64.

---

## Part 1: Design decisions

### Native ARM64 over emulated x86
The first SIEM VM was built as an emulated x86 machine (UTM "Standard PC / Q35"). On Apple Silicon this means full software emulation, which is slow, and the Wazuh indexer is resource-heavy. That VM also corrupted its filesystem twice after being force-stopped.

**Decision:** rebuild every VM as native ARM64 using UTM's "Virtualize" option. Wazuh 4.x fully supports ARM64 for all components. Result: dramatically faster and stable.

**Lesson:** match the guest architecture to the host. Emulation is a last resort, not a default.

### Isolated network
All VMs sit on one internal/host-only network so attack traffic and any future malware analysis stay contained and cannot reach the host network or internet.

---

## Part 2: Issues encountered and resolutions

### Issue 1: VM filesystem corruption after force-stop

**Symptom:** VM dropped to an `(initramfs)` emergency shell on boot with:
```
UNEXPECTED INCONSISTENCY; RUN fsck MANUALLY.
Failure: File system check of the root filesystem failed
```

**Cause:** the VM had been stopped using UTM's stop button rather than a clean shutdown, corrupting the filesystem mid-write. Emulated VMs are also more fragile.

**Resolution:** ran a manual filesystem check from the recovery shell:
```
fsck -y /dev/mapper/ubuntu--vg-ubuntu--lv
```
When corruption recurred, the durable fix was to abandon the emulated VM and rebuild native ARM64.

**Lesson:** always shut a VM down cleanly (`sudo poweroff`), never the hypervisor stop button. Snapshot or clone working states so recovery takes minutes, not hours.

---

### Issue 2: Wazuh dashboard component missing

**Symptom:** after install, the browser could not reach the dashboard (`ERR_CONNECTION_REFUSED`), and on the server:
```
Unit wazuh-dashboard.service could not be found.
```

**Cause:** an earlier interrupted install left the manager and indexer in place but never installed the dashboard component.

**Resolution:** rather than patch a partial install, ran a clean full reinstall that regenerates all components and their certificates together:
```
sudo bash wazuh-install.sh -a -o
```
The `-o` (overwrite) flag tears down the partial install and produces a complete, correctly wired stack, printing a fresh admin password in the summary.

**Lesson:** for a multi-component install left in a broken partial state, a clean overwrite is more reliable than trying to bolt on the missing piece.

---

### Issue 3: Lost admin password

**Symptom:** the install-files archive containing the generated admin password could not be found, so dashboard login was impossible.

**Resolution:** reset the admin password directly with Wazuh's password tool:
```
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin -p 'NewPassword!'
```
Password policy required 8-64 characters with upper, lower, number, and symbol. The `-u` flag is the account (must be `admin`), `-p` is the new password.

**Lesson:** understand the difference between the flags. An early mistake put the password value in the `-u` (username) slot, producing "the given user does not exist." Read the tool's parameters rather than pattern-matching.

---

### Issue 4: Browser could not reach the dashboard (HTTPS)

**Symptom:** connection refused even after the dashboard service was confirmed running.

**Cause:** the browser defaulted to plain `http`. Wazuh only serves over `https`.

**Resolution:** access explicitly via `https://<server-ip>` and accept the self-signed certificate warning (expected in a lab).

**Diagnostic approach:** confirmed the service was listening before assuming a network fault:
```
sudo ss -tlnp | grep 443
sudo systemctl status wazuh-dashboard
```
Isolating "is the service up?" from "can I reach it?" narrows the problem quickly.

---

### Issue 5: Agent install blocked by broken package state (the hardest one)

**Symptom:** installing the Wazuh agent on the endpoint failed repeatedly:
```
dpkg: error processing archive ... pre-installation script subprocess returned error exit status 127
```
A previous agent install had left the package in a broken half-installed state that blocked the new install, and normal removal also failed because the old package's own removal scripts errored out. A classic dependency deadlock.

**Resolution (step by step):**
1. Neutralised the broken maintainer scripts so dpkg could run them without error:
   ```
   for f in /var/lib/dpkg/info/wazuh-agent.*; do echo -e '#!/bin/sh\nexit 0' | sudo tee "$f" >/dev/null; done
   ```
2. Removed leftover metadata:
   ```
   sudo rm -f /var/lib/dpkg/info/wazuh-agent.*
   ```
3. Force-removed the package, which moved it from a broken `pi` (half-installed) state to `rc` (removed, config remaining):
   ```
   sudo dpkg --remove --force-all wazuh-agent
   ```
4. Cleared the remaining config with a purge:
   ```
   sudo dpkg --purge wazuh-agent
   ```
5. Confirmed the package was fully gone (`dpkg -l | grep wazuh` returned nothing), then reinstalled cleanly.

**Lesson:** understanding dpkg package states (`ii`, `pi`, `rc`) and where dpkg stores its metadata (`/var/lib/dpkg/info/`) turns an unrecoverable-looking error into a methodical fix. This mirrors real endpoint remediation work.

---

### Issue 6: SSH not present on the endpoint

**Symptom:** the brute-force target had no SSH service:
```
Unit ssh.service could not be found.
```

**Cause:** the desktop Ubuntu build did not include the SSH server, which the attack requires as a target.

**Resolution:**
```
sudo apt install openssh-server -y
sudo systemctl enable --now ssh
```

**Lesson:** confirm the attack surface exists before attacking it. `enable --now` both starts a service and sets it to start on boot.

---

## Part 3: Verification discipline

A recurring theme: verify each layer before building the next.

- After the SIEM install, confirmed the dashboard loaded before adding endpoints.
- After agent install, confirmed `active (running)` before checking the dashboard.
- Before running the attack, confirmed SSH was listening and auditd was capturing events locally.
- Distinguished the SIEM server (192.168.64.16) from the endpoint (192.168.64.8) when reading status, an agent service does not run on the server, so "unit not found" there is correct, not an error.

Verifying each layer means any failure has one likely cause instead of several. This is the single habit that made the build tractable.

---

## Part 4: Telemetry and detection

- Installed auditd on the endpoint and added a watch rule on `/etc/passwd` (key `identity_change`) for identity-related changes.
- Confirmed capture locally with `ausearch -k identity_change` before checking forwarding.
- Ran a controlled SSH brute force from Kali with Hydra (small wordlist, `-t 4` parallelism) to trigger detection without overwhelming the target.
- Wazuh correlated the repeated authentication failures and raised an alert mapped to MITRE ATT&CK T1110 (Brute Force / Password Guessing).
- Triaged the alert to source IP, targeted account, rule level, fired-times count, and the raw log line.

---

## Summary of skills demonstrated through troubleshooting

- Linux filesystem recovery (`fsck`)
- Package management internals and recovery (dpkg states, metadata, force operations)
- Service and port diagnostics (`systemctl`, `ss`)
- Credential management and reset tooling
- Methodical fault isolation across a multi-component system
- Clean operational habits (safe shutdown, snapshots, layer-by-layer verification)
