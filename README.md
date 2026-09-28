# SMB Brute-Force Detection Lab — Kali Linux → Windows 11 → Wazuh

A hands-on home lab exercise simulating a credential brute-force attack against a Windows 11 host, with detection and analysis performed through a Wazuh SIEM deployment.

## Lab Environment

| Role | VM | Details |
|---|---|---|
| Attacker | Kali Linux 2026.1 | VMware Workstation guest |
| Victim | Windows 11 x64 (Build 26100) | Sysmon + NXLog + Wazuh agent installed |
| SIEM / Log Server | Ubuntu Server | Wazuh manager, rsyslog, Suricata |

All three VMs run on the same virtual network segment (`192.168.203.0/24`) within VMware Workstation.

> 📸 **Screenshot placeholder:** network topology diagram or VMware Workstation library view showing all three VMs.

## Objective

Simulate an SMB credential brute-force attack against the Windows 11 victim and validate whether the monitoring stack (Sysmon → NXLog → rsyslog → Wazuh) detects, logs, and correlates the activity.

## Methodology

### 1. Recon — confirm SMB dialect support

```bash
nmap -Pn -p 445 --script smb-protocols 192.168.203.140
```

Confirmed the target supports **SMB dialects 2.0.2 through 3.1.1** with SMBv1 disabled — standard for Windows 10/11 since ~2017. This determined the choice of attack tool below (legacy SMBv1-only tools will not work against this target).

> 📸 **Screenshot placeholder:** `nmap smb-protocols` output showing the dialect list.

### 2. Attack execution — netexec (SMB2/3-capable)

```bash
netexec smb 192.168.203.140 -u admin -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding
```

- `--ignore-pw-decoding` skips non-UTF-8 lines in `rockyou.txt`.
- netexec fingerprinted the target on connect: `Windows 11 / Server 2025 Build 26100, signing:True, SMBv1:None`.

> 📸 **Screenshot placeholder:** netexec terminal output showing the connection/fingerprint line and attempt results.

## Detection Results (Wazuh)

The attack was fully visible in the Wazuh Threat Hunting view for the `windows11pro` agent:

- **Individual failed logons** — dozens of `Logon Failure - Unknown user or bad password` events (Windows Event ID 4625), timestamped in rapid succession (sub-second intervals), forwarded via Sysmon/NXLog through rsyslog to the Wazuh manager.
- **Correlated alert** — Wazuh's built-in brute-force detection rule fired a `Multiple Windows Logon Failures` alert (rule level 10) once the failed-logon threshold was crossed within its detection window.
- **Baseline noise** — routine `A process was created` Sysmon events continued throughout, useful as a reference for distinguishing attack-related spikes from normal system activity.
- **SCA (Security Configuration Assessment) findings** — the CIS Microsoft Windows 11 Enterprise Benchmark v3.0.0 scan flagged the host's overall compliance score at **under 30%**, specifically citing:
  - `Account lockout duration` not meeting the recommended 15+ minute minimum
  - `Enforce password history` not meeting the recommended 24-password minimum

> 📸 **Screenshot placeholder:** Wazuh Threat Hunting timeline showing the run of `Logon Failure` events and the `Multiple Windows Logon Failures` correlated alert.

> 📸 **Screenshot placeholder:** Wazuh SCA summary panel showing the CIS benchmark score and the two flagged policy checks.

## Findings

1. **Modern SMB stacks require modern tooling** — a dialect check up front (`nmap --script smb-protocols`) is a necessary step before choosing an attack tool; legacy SMBv1-only tooling will silently fail against Windows 10/11 defaults.
2. **netexec successfully executed the brute-force** against SMB2/3 with signing enabled, confirming that dialect version alone is not a protection against credential attacks — only account lockout policy, strong passwords, or MFA/alternative auth actually stop this class of attack.
3. **No effective account lockout observed** — the attack ran for an extended volume of attempts without a clear lockout response, consistent with the CIS benchmark finding that lockout duration policy was not properly configured.
4. **Detection worked as intended** — Wazuh correctly ingested Windows Event ID 4625s via the Sysmon/NXLog/rsyslog pipeline and correlated them into a single actionable alert, validating the monitoring pipeline built in this lab.

## Recommendations (Windows Hardening)

| Area | Current State | Recommendation |
|---|---|---|
| Account lockout duration | Not meeting CIS benchmark (<15 min) | Set lockout duration to 15+ minutes via `Local Security Policy → Account Lockout Policy` or GPO |
| Account lockout threshold | Not effectively limiting repeated attempts | Set a lockout threshold (e.g., 5–10 invalid attempts) to stop sustained brute-force runs |
| Password history | Not meeting CIS benchmark (<24 passwords) | Set "Enforce password history" to 24 or more passwords |
| SMB signing | Enabled (good) | Keep enabled; also consider requiring SMB signing on both client and server (`Microsoft network server: Digitally sign communications (always)`) |
| SMBv1 | Disabled (good) | No action — confirm it stays disabled; do not re-enable for compatibility without strong justification |
| Credential strength | Default/weak lab credentials used | Enforce strong password policy and consider MFA for any exposed authentication service |
| Monitoring | Working correctly | Continue tuning Wazuh's brute-force correlation rule threshold/window to balance detection speed vs. false positives |

## Next Steps

- Inspect the specific Wazuh rule ID/threshold behind "Multiple Windows Logon Failures" to understand exact sensitivity.
- Apply the account lockout policy fix and re-run the attack to confirm the lockout now interrupts the brute-force attempt.
- Extend the exercise with Suricata to check whether the SMB traffic pattern itself (not just Windows event logs) triggers a network-layer alert.

## Disclaimer

This exercise was performed entirely within an isolated home lab against VMs owned and controlled by the author. No attack techniques described here should be used against systems without explicit authorization.
