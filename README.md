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
## Recommended Mitigation

1. **Configure Default Firewall Policies**

   Set the default incoming traffic policy to deny and outgoing traffic policy to allow:

   ```bash
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   ```

2. **Allow Authorized Administrative Access**

   Explicitly permit SSH connections on TCP port 22:

   ```bash
   sudo ufw allow ssh
   ```

3. **Enable the Firewall**

   Enable UFW to ensure firewall protection remains active across system reboots:

   ```bash
   sudo ufw --force enable
   ```

---

## Finding 02: Unhardened OpenSSH Authentication Directives

- **Affected Component:** OpenSSH Server Configuration (`/etc/ssh/sshd_config`)
- **Risk Level:** Medium

### Description

The core SSH authentication directives (`PermitRootLogin`, `PasswordAuthentication`, and `PermitEmptyPasswords`) are commented out in the OpenSSH server configuration file, relying on package defaults.

Specifically, the commented directive `#PasswordAuthentication yes` indicates that password-based authentication may remain enabled depending on the effective SSH configuration. This increases the host's exposure to password-based brute-force attacks.

### Evidence

**Evidence File:** `evidence/02_sshd_audit.txt`

```bash
$ sudo grep -Ei "PermitRootLogin|PasswordAuthentication|PermitEmptyPasswords" /etc/ssh/sshd_config | tee evidence/02_sshd_audit.txt

#PermitRootLogin prohibit-password
#PasswordAuthentication yes
#PermitEmptyPasswords no
```

 ### Recommended Mitigation

1. Explicitly configure `PasswordAuthentication no` in `/etc/ssh/sshd_config` to enforce key-based SSH authentication.
2. Explicitly configure the following security directives:

   ```bash
   PermitRootLogin no
   PasswordAuthentication no
   PermitEmptyPasswords no
   ```

3. Validate the SSH configuration syntax before applying changes:

   ```bash
   sudo sshd -t
   ```

4. If the configuration passes validation, reload the SSH service:

   ```bash
   sudo systemctl reload ssh
   ```

---

## Finding 03: World-Readable SSH Daemon Configuration and Core File Permissions Audit

- **Affected Component:** File System Access Controls (`/etc/ssh/sshd_config`, `/etc/shadow`, `/etc/passwd`, `/etc/group`)
- **Risk Level:** Low / Medium

### Description

The permissions audit identified that `/etc/shadow` was properly restricted to the `root:shadow` ownership group with permissions set to `0640`.

However, `/etc/ssh/sshd_config` was configured with permissions of `0644` (`-rw-r--r--`), allowing all local users to read the SSH daemon configuration.

Although SSH configuration files generally do not contain authentication secrets, restricting access can reduce unnecessary local reconnaissance and improve overall system hardening.

### Evidence

**Evidence File:** `evidence/04_permissions_audit.txt`

```bash
$ ls -l /etc/passwd /etc/shadow /etc/group /etc/ssh/sshd_config | tee evidence/04_permissions_audit.txt

-rw-r--r-- 1 root root   1315 Nov  9 20:42 /etc/group
-rw-r--r-- 1 root root   3263 Nov  9 20:42 /etc/passwd
-rw-r----- 1 root shadow 1464 Nov  9 20:42 /etc/shadow
-rw-r--r-- 1 root root   3424 Jun  1 06:37 /etc/ssh/sshd_config
```

### Recommended Mitigation

1. Restrict access to `/etc/ssh/sshd_config` so that only the root user can read and modify the file.

   ```bash
   sudo chmod 600 /etc/ssh/sshd_config
   sudo chown root:root /etc/ssh/sshd_config
   ```

2. Verify the updated permissions:

   ```bash
   ls -l /etc/ssh/sshd_config
   ```

---

## Finding 04: Baseline Network Socket Enumeration

- **Affected Component:** Local TCP/UDP Network Stack (`ss -tulpn`)
- **Risk Level:** Informational (Baseline Control)

### Description

A network socket enumeration was performed using the `ss -tulpn` command to identify active TCP and UDP listening services.

The assessment confirmed that no unnecessary or unauthorized network services were actively listening on network interfaces at the time of the audit.

This represents a minimal network service footprint and reduces the host's potential attack surface.

### Evidence

**Evidence File:** `evidence/01_listening_ports.txt`

```bash
$ sudo ss -tulpn | tee evidence/01_listening_ports.txt

Netid State Recv-Q Send-Q Local Address:Port Peer Address:Port Process
```

### Recommended Mitigation

1. Maintain a minimal network service footprint.
2. Disable unnecessary services through `systemd`.
3. Periodically review listening network sockets:

   ```bash
   sudo ss -tulpn
   ```

---

## 5. Remediation Verification (Host Firewall Hardening)

To address **Finding 01**, the Uncomplicated Firewall (UFW) was configured with a default-deny incoming policy and a default-allow outgoing policy.

SSH access was explicitly permitted through TCP port 22, and the firewall was enabled to provide persistent protection across system reboots.

### Verification Evidence

**Evidence File:** `evidence/05_remediation_verified.txt`

```bash
$ sudo ufw status verbose | tee evidence/05_remediation_verified.txt

Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
```

### Verification Results

| Security Control | Expected Configuration | Status |
|---|---|---|
| Firewall Status | Active | Verified |
| Incoming Traffic | Default Deny | Verified |
| Outgoing Traffic | Default Allow | Verified |
| SSH Access | TCP Port 22 Allowed | Verified |
| IPv6 SSH Access | TCP Port 22 Allowed | Verified |
| Firewall Logging | Enabled (Low) | Verified |

---

## 6. Authorization and Ethics Disclaimer

All security assessments, configuration reviews, and system hardening activities documented in this repository were performed strictly within an isolated, self-owned Kali Linux virtual machine environment.

These activities were conducted in accordance with SWYNEX Technologies internship guidelines and were limited to authorized systems.

No third-party systems, networks, or infrastructure were targeted during this assessment.

The purpose of this exercise was to identify potential security misconfigurations, implement appropriate remediation measures, and verify the effectiveness of the applied security controls.

---

## 7. Evidence and Screenshots

Supporting screenshots and audit logs are maintained in the `evidence/` directory.

The evidence files provide documentation of the initial security findings, configuration changes, and remediation verification results.

**Note:** If screenshots are stored directly in the repository's root directory instead of the `evidence/` directory, update the corresponding image paths in this README by removing the `evidence/` prefix.
