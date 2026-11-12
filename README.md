# SWYNEX Security Fundamentals Assessment (Task 1)

**Intern:** Anuj Agrawal  
**Intern ID:** SWX-APP-2026-9605F  
**Domain:** Cloud & Emerging Technology (Cyber Security)  
**Environment:** Authorized Local Virtualized Lab (Kali Linux VM — Host: `kali`)  
**Date:** November 12, 2026  

---

## 1. Executive Summary
This repository documents a baseline Security Fundamentals Assessment conducted on an authorized, isolated Kali Linux virtual machine (`kali@kali`). The assessment evaluated active network sockets, OpenSSH daemon authentication parameters, host-based firewall status (`ufw`), and critical file permissions. Key risks identified included an inactive host-based firewall, implicit password authentication defaults in SSH, and world-readable SSH daemon configuration files. Host firewall remediation was executed and verified during the assessment.

## 2. Scope & Methodology
* **Target System:** Local Kali Linux Virtual Machine (`kali@kali`)
* **Assessment Utilities:** `ss`, `grep`, `ufw`, `ls` (POSIX file permission inspection)
* **Methodology:** Local socket enumeration, daemon configuration auditing, host firewall verification, privilege/permission inspection, and active remediation verification.

---

## 3. Security Findings Summary

| ID | Affected Component | Risk Level | Summary of Finding | Status |
| :--- | :--- | :--- | :--- | :--- |
| **F-01** | Host Firewall (`ufw`) | **High** | Host-based firewall disabled (`Status: inactive`) with no ingress filtering | **Remediated** |
| **F-02** | OpenSSH Daemon (`/etc/ssh/sshd_config`) | **Medium** | Explicit hardening directives commented out; `PasswordAuthentication` defaults to `yes` | Documented |
| **F-03** | SSH Configuration File Permissions | **Low / Med** | `/etc/ssh/sshd_config` is world-readable (`-rw-r--r--` / `0644`) to unprivileged local users | Documented |
| **F-04** | Network Socket Surface (`ss -tulpn`) | **Informational** | Zero unauthorized listening TCP/UDP daemons enabled by default | Verified Clean |

---

## 4. Detailed Findings, Evidence & Recommended Mitigations

### Finding 01: Disabled Host-Based Firewall (UFW)
* **Affected Component:** Uncomplicated Firewall (`ufw` / Netfilter ingress filtering)
* **Risk:** **High** — With the host firewall inactive, the system lacks network-layer ingress filtering. Any service started temporarily or bound to `0.0.0.0` is immediately reachable by adjacent hosts on the local network segment.
* **Evidence:**
  * Log file: `evidence/03_firewall_audit.txt`
  ```text
  $ sudo ufw status verbose | tee evidence/03_firewall_audit.txt
  Status: inactive
