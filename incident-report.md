# Incident Report: Simulated Attack Detection & Vulnerability Assessment

**Analyst:** Paul
**Date:** August 11, 2026
**Environment:** Self-hosted Wazuh SIEM lab

---

## Summary

On August 11, 2026, a series of simulated attack techniques was conducted against a monitored Ubuntu endpoint (`victim-vm`) as part of a self-directed SOC lab exercise. Activity included SSH password-guessing attempts, sudo/privilege-escalation actions, and unauthorized user account creation, all originating from `192.168.1.161`. The Wazuh SIEM detected and logged all activity in near real time, correctly mapping it to multiple MITRE ATT&CK techniques and tactics. Notably, sustained password-guessing activity escalated to a **high-severity (Level 10) correlated brute-force alert**, demonstrating Wazuh's pattern-based detection in addition to single-event logging. A supplementary vulnerability scan of the endpoint was also performed using Wazuh's built-in Vulnerability Detection module.

## Environment

| Item | Value |
|---|---|
| SIEM | Wazuh 4.14.7 (self-hosted, OVA deployment) |
| Monitored endpoint | `victim-vm` (Ubuntu 26.04 LTS, agent ID 001) |
| Endpoint IP | 192.168.1.163 |
| Attack source | 192.168.1.161 |
| Detection window | 2026-08-10 13:29 UTC – 2026-08-11 13:37 UTC |

## Timeline

1. Attacker (host machine) initiated multiple manual SSH connection attempts against `victim-vm`, using both non-existent usernames and incorrect passwords for valid accounts.
2. Repeated authentication failures within a short window triggered Wazuh's correlated brute-force detection rule.
3. A successful sudo escalation to root was performed on `victim-vm`, followed by creation of a new local user account (`suspicious_user`) to simulate a persistence/backdoor-account technique.
4. The Wazuh agent forwarded all corresponding log events (journald, PAM, sshd, useradd) to the manager in near real time, where the rule engine matched them against its detection ruleset and auto-mapped them to MITRE ATT&CK.
5. A vulnerability scan of installed packages on `victim-vm` was run via Wazuh's Vulnerability Detection module.

## Detection: Alerts Mapped to MITRE ATT&CK

The table below reflects alerts filtered specifically to those with a MITRE ATT&CK mapping (`rule.mitre.id: *`), isolating attacker-relevant activity from routine baseline/compliance scans:

| Rule ID | Description | Level | Count | MITRE Technique | MITRE Tactic |
|---|---|---|---|---|---|
| 5710 | sshd: Attempt to login using a non-existent user | 5 | 10 | Password Guessing (T1110.001) | Credential Access |
| 5501 | PAM: Login session opened | 3 | 5 | SSH / Valid Accounts | Credential Access / Lateral Movement |
| 5402 | Successful sudo to ROOT executed | 3 | 3 | Sudo and Sudo Caching | Privilege Escalation |
| 5503 | PAM: User login failed | 5 | 3 | Password Guessing (T1110.001) | Credential Access |
| 5557 | unix_chkpwd: Password check failed | 5 | 3 | Password Guessing (T1110.001) | Credential Access |
| 5403 | First time user executed sudo | 4 | 2 | Sudo and Sudo Caching | Privilege Escalation |
| **2502** | **syslog: User missed the password more than one time** | **10** | 1 | **Brute Force (T1110)** | **Credential Access** |
| **5902** | **New user added to the system** | **8** | 1 | **Create Account (T1136)** | **Persistence** |

**Total MITRE-mapped alerts:** 28
**Tactics observed:** Credential Access, Privilege Escalation, Persistence (directly attacker-driven); Defense Evasion, Lateral Movement, Initial Access also appear in the dashboard's tactic breakdown but stem from Wazuh's default technique-to-tactic mapping for standard SSH session/login events rather than distinct attacker actions beyond what's listed above — see Analysis.

### Highlighted alert 1 — Correlated brute-force escalation (Rule 2502, Level 10)

This is the strongest finding in the exercise: rather than logging each failed attempt in isolation, Wazuh correlated repeated authentication failures from the same source into a single higher-severity alert.

```
Aug 11 17:27:09 paul-ubuntu sshd-session[7089]: PAM 2 more authentication failures;
logname= uid=0 euid=0 tty=ssh ruser= rhost=192.168.1.161
```
- **MITRE:** T1110 (Brute Force), Tactic: Credential Access
- **Compliance mapping:** PCI DSS 10.2.4/10.2.5, HIPAA 164.312.b, NIST 800-53 AU.14/AC.7, GDPR IV_35.7.d/IV_32.2

### Highlighted alert 2 — Unauthorized account creation (Rule 5902, Level 8)

```
Aug 11 17:33:05 paul-ubuntu useradd[7174]: new user: name=suspicious_user, UID=1001,
GID=1001, home=/home/suspicious_user, shell=/bin/sh, from=/dev/pts/2
```
- **MITRE:** T1136 (Create Account), Tactic: Persistence
- **Compliance mapping:** PCI DSS 10.2.7/10.2.5/8.1.2, HIPAA 164.312.b/164.312.a.2.I/II, NIST 800-53 AU.14/AC.7/AC.2/IA.4

## Vulnerability Assessment

Wazuh's Vulnerability Detection module was used to passively scan installed packages on `victim-vm` (Ubuntu 26.04 LTS) against known CVE databases — no exploitation was performed, this reflects standard package-level exposure.

| Severity | Count |
|---|---|
| Critical | 67 |
| High | 457 |
| Medium | 415 |
| Low | 69 |
| Pending Evaluation | 311 |
| **Total findings** | **1,319** |

**Top identified CVEs:** CVE-2023-3326 (16 occurrences), CVE-2026-27456 (14), CVE-2026-3184 (14), CVE-2026-40228 (11), CVE-2017-13716 (8)

**Top affected packages:** `linux-image-7.0.0-29-generic` (719), `firefox` (126), `rust-coreutils` (20), `bluez` (19), `bluez-cups` (19)

The majority of findings are concentrated in the kernel image package, consistent with a stock Ubuntu install that has not yet been patched to the latest point release.

## Analysis

The presence of login attempts against non-existent usernames, combined with the escalation to a correlated Level 10 brute-force alert, is a strong indicator of automated or sustained credential-guessing behavior rather than a legitimate user's typo. Unlike the first test pass in this exercise, this run generated sufficient volume within a short enough window to cross Wazuh's correlation threshold — demonstrating the difference between simple log-matching detection and pattern/frequency-based correlation, which is a materially more sophisticated detection capability.

The subsequent sudo escalation and new-user creation extend the exercise beyond credential access into privilege escalation and persistence, giving broader technique coverage across the MITRE ATT&CK matrix than authentication failures alone.

**Accuracy note:** the dashboard's "Top tactics" view also surfaces Defense Evasion, Lateral Movement, and Initial Access. These stem from Wazuh's default rule-to-MITRE mapping applied to standard SSH session and login events (e.g., a normal session open is tagged under "Valid Accounts"/"SSH" techniques, which map to those tactics in Wazuh's ruleset) rather than distinct attacker actions performed during this exercise. Distinguishing genuinely tested techniques from auto-tagged routine activity is called out here for accuracy.

## Recommendations

1. Disable password-based SSH authentication in favor of key-based authentication.
2. Deploy fail2ban (or equivalent) to automatically block source IPs after repeated authentication failures — this exercise's Level 10 alert is exactly the kind of event such a tool would act on.
3. Restrict SSH access to known management IP ranges via host or network firewall rules.
4. Enable Wazuh active response to automatically block offending source IPs after a defined failure threshold.
5. Audit and remove the `suspicious_user` test account; in production, alert on any new local account creation outside change-management processes.
6. Prioritize patching for the kernel image package and Firefox given their disproportionate share of vulnerability findings; address the 67 critical-severity CVEs first.

## Appendix

- Wazuh Threat Hunting Report (PDF), full detection window.
- Wazuh MITRE ATT&CK Report (PDF), tactic/technique breakdown.
- Wazuh Vulnerability Detection module screenshot, `victim-vm` package scan results.
- Raw alert JSON for rules 2502 and 5902, Wazuh dashboard Discover view.
