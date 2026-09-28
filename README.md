# SMB Brute-Force Detection Lab — Kali Linux → Windows 11 → Wazuh

A hands-on home lab exercise simulating a credential brute-force attack against a Windows 11 host, with detection and analysis performed through a Wazuh SIEM deployment.

## Lab Environment

| Role | VM | Details |
|---|---|---|
| Attacker | Kali Linux 2026.1 | VMware Workstation guest |
| Victim | Windows 11 x64 (Build 26100) | Sysmon + NXLog + Wazuh agent installed |
| SIEM / Log Server | Ubuntu Server | Wazuh manager, rsyslog, Suricata |
<br>


**All three VMs** run on the same virtual network segment (`192.168.203.0/24`) within VMware Workstation.

> <img width="400" height="250" alt="Screenshot 2026-08-29 125326" src="https://github.com/user-attachments/assets/6a4ba2cd-e227-47a1-a351-40925ed3d0d4" />


## Objective

Simulate an SMB credential brute-force attack against the Windows 11 victim and validate whether the monitoring stack (Sysmon → NXLog → rsyslog → Wazuh) detects, logs, and correlates the activity.

## Methodology

### 1. Attack execution — netexec (SMB2/3-capable)

netexec smb 192.168.203.140 -u admin -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding

- `--ignore-pw-decoding` skips non-UTF-8 lines in `rockyou.txt`.
- netexec fingerprinted the target on connect: `Windows 11 / Server 2025 Build 26100, signing:True, SMBv1:None`.

>  <img width="540" height="250" alt="SMB output" src="https://github.com/user-attachments/assets/a5d51f64-279e-451c-a408-eedcfef78003" />


## Detection Results (Wazuh)

The attack was fully visible in the Wazuh Threat Hunting view for the `windows11pro` agent:

- **Individual failed logons** — dozens of `Logon Failure - Unknown user or bad password` events (Windows Event ID 4625), timestamped in rapid succession (sub-second intervals), forwarded via Sysmon/NXLog through rsyslog to the Wazuh manager.
- **Correlated alert** — Wazuh's built-in brute-force detection rule fired a `Multiple Windows Logon Failures` alert (rule level 10) once the failed-logon threshold was crossed within its detection window.
- **Baseline noise** — routine `A process was created` Sysmon events continued throughout, useful as a reference for distinguishing attack-related spikes from normal system activity.
- **SCA (Security Configuration Assessment) findings** — the CIS Microsoft Windows 11 Enterprise Benchmark v3.0.0 scan flagged the host's overall compliance score at **under 30%**, specifically citing:
  - `Account lockout duration` not meeting the recommended 15+ minute minimum
  - `Enforce password history` not meeting the recommended 24-password minimum


**Wazuh Threat Hunting timeline showing the run of `Logon Failure` events and the `Multiple Windows Logon Failures` correlated alert.**

<img width="600" height="320" alt="logon failures" src="https://github.com/user-attachments/assets/1acf20fe-c4b0-4395-ac62-318eef1081c7" />
<br>


**Wazuh SCA summary panel showing the CIS benchmark score and the two flagged policy checks.**


<img width="600" height="320" alt="CIS bench mark" src="https://github.com/user-attachments/assets/a53a7a67-c8c4-424e-bf73-3628749e3e35" />



## Findings

1. **netexec successfully executed the brute-force** against SMB2/3 with signing enabled, confirming that dialect version alone is not a protection against credential attacks — only account lockout policy, strong passwords, or MFA/alternative authentication actually stop this class of attack.
2. **No effective account lockout observed** — the attack ran for an extended volume of attempts without a clear lockout response, consistent with the CIS benchmark finding that lockout duration policy was not properly configured.
3. **Detection worked as intended** — Wazuh correctly ingested Windows Event ID 4625s via the Sysmon/NXLog/rsyslog pipeline and correlated them into a single actionable alert, validating the monitoring pipeline built in this lab.

## Recommendations (Windows Hardening)

| Area | Current State | Recommendation |
|---|---|---|
| Account lockout duration | Not meeting CIS benchmark (<15 min) | Set lockout duration to 15+ minutes via `Local Security Policy → Account Lockout Policy` or GPO |
| Account lockout threshold | Not effectively limiting repeated attempts | Set a lockout threshold (e.g., 5–10 invalid attempts) to stop sustained brute-force runs |
| Password history | Not meeting CIS benchmark (<24 passwords) | Set "Enforce password history" to 24 or more passwords |
| SMB signing | Enabled (good) | Keep enabled; also consider requiring SMB signing on both client and server (`Microsoft network server: Digitally sign communications (always)`) |
| Credential strength | Default/weak lab credentials used | Enforce strong password policy and consider MFA for any exposed authentication service |
| Monitoring | Working correctly | Continue tuning Wazuh's brute-force correlation rule threshold/window to balance detection speed vs. false positives |
<br>


**Account Lockout Duration**

<img width="600" height="320" alt="Account lockout duration" src="https://github.com/user-attachments/assets/877840d7-861d-4000-b441-1d2b27569c34" />
<br>


**Password History Setting**


<img width="600" height="320" alt="Password history" src="https://github.com/user-attachments/assets/2e61252e-3eab-45cd-864d-3fe673f312ff" />


