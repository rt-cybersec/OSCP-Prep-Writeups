# GoldenEye v1: Walkthrough

## Overview
**Machine Name**: GoldenEye v1
**Platform**: VulnHub
**Difficulty**: Medium
**OS**: Linux (Ubuntu 14.04)
**Focus**: SMTP/POP3 Credential Enumeration, Moodle CMS Exploitation, Local Privilege Escalation (Overlayfs Exploit)

## Enumeration
### Network Discovery
An ARP sweep located the target machine at IP address `192.168.29.250`.
```bash
arp-scan -l -I eth1
```

### Port Scanning
Nmap service detection was executed on the target host:
The scan revealed POP3 mail services on ports `55006` / `55007` and an HTTP web server on port `80`.

### Web Enumeration
Accessing the HTTP service on port 80 showed a landing page. Examining the Javascript source files revealed an admin/developer comment displaying the user credentials:
`Boris` : `InvincibleHack3r`.

### POP3 Enumeration & Mail Access
Connecting to POP3 on port `55007` was tested with `boris:InvincibleHack3r` but failed.
A Hydra credential brute-force attack was run against POP3 to test user names and passwords from the default lists:
```bash
hydra -l boris -P /usr/share/wordlists/fasttrack.txt -t20 192.168.29.250 -s55007 -I pop3
```
This discovered valid login credentials:
- `boris` : `secret1!`
- `natalya` : `bird`

Logging into POP3 mail server using `natalya:bird` and `boris:secret1!` allowed reading internal emails.
An email in Natalya's mailbox mentioned a Moodle installation path and credentials for `xenia`:
`xenia` : `RCP90rulez!`.

In another email, credentials for `doak` were cracked via POP3:
`doak` : `goat`.

Using POP3, the user `dr_doak` mail account was read, disclosing Moodle admin-level permissions and password:
`dr_doak` : `4England!`.

## Exploitation
### Moodle RCE
Logging into Moodle using `dr_doak:4England!` allowed access to the site administration panel. 
The system configuration allowed configuring the path to the system spellchecker (aspell). 

A reverse shell payload was uploaded, and the spellchecker path was set to call the reverse shell binary. Triggering a spell check generated a connection back to the attacker host.
```bash
whoami
www-data
```

## Privilege Escalation
### Local Kernel Enumeration
The system operating version was checked:
```bash
cat /etc/os-release
NAME="Ubuntu"
VERSION="14.04.1 LTS, Trusty Tahr"
```

### Overlayfs Privilege Escalation
An exploit for the local overlayfs kernel vulnerability was found (CVE-2015-1328 / exploit 37292).
Since `gcc` was not installed on the system, the C source file was edited to replace the compilation references from `gcc` to `cc`, compiling successfully on the target machine:
```bash
cc exploit.c -o exploit
./exploit
```
![](screenshots/Pasted%20image%2020260527205917.png)
```bash
whoami
root
```

## Post-Exploitation
### Flag Recovery
- Root Flag obtained after spawning the root shell.
  ![](screenshots/Pasted%20image%2020260527205917.png)

## Lessons Learned
1. Do not leave developer credentials in plaintext inside Javascript source files.
2. Ensure user inputs on spellchecker binaries or execution paths in CMS platforms are strictly validated and sanitized.
3. Keep the base Linux operating system kernel updated to patch well-known local privilege escalation vulnerabilities.
