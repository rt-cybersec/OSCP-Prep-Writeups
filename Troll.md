# Troll: Walkthrough

## Overview
**Machine Name**: Troll<br>
**Platform**: VulnHub<br>
**Difficulty**: Easy<br>
**OS**: Linux (Ubuntu)<br>
**Focus**: FTP PCAP Extraction, Username/Password Spraying, SUID Privilege Escalation

## Enumeration

### Port Scanning
An Nmap service scan discovered open ports:
![](screenshots/Pasted%20image%2020260419120725.png)
Open services:
- 21/tcp (FTP)
- 22/tcp (SSH)
- 80/tcp (HTTP)

### Web & FTP Enumeration
Accessing the HTTP service on port 80 returned a default website containing a troll image:
![](screenshots/Pasted%20image%2020260419120840.png)

Directories `/secret` and `/hacker.jpg` displayed similar troll images:
![](screenshots/Pasted%20image%2020260419121258.png)
![](screenshots/Pasted%20image%2020260419121345.png)

Anonymous FTP access was enabled on port 21. Connecting anonymously allowed downloading a network packet capture file named `lol.pcap`:
![](screenshots/Pasted%20image%2020260419121500.png)

The capture file `lol.pcap` was analyzed in Wireshark. Filtering for FTP traffic (`ftp-data`) revealed a hidden directory string:
`sup3rS3cr3td1r3ct0ry`
![](screenshots/Pasted%20image%2020260419121633.png)

Navigating to the directory `/sup3rS3cr3td1r3ct0ry` on the HTTP server:
![](screenshots/Pasted%20image%2020260419121752.png)

Inside the directory, an ELF binary executable named `roflmao` was downloaded:
![](screenshots/Pasted%20image%2020260419121902.png)

Executing the `roflmao` binary locally outputted a memory address clue:
`0x0856bf`
![](screenshots/Pasted%20image%2020260419121936.png)

Browsing to the directory `/0x0856bf`:
![](screenshots/Pasted%20image%2020260419122016.png)
This directory contained two subfolders:
- `/good_luck` (containing a text file with a list of system usernames)
  ![](screenshots/Pasted%20image%2020260419122046.png)
- `/this_is_not_a_password` (containing `Pass.txt` with a list of passwords)
  ![](screenshots/Pasted%20image%2020260419122131.png)

## Exploitation
### SSH Password Spraying
A credential spray attack was executed using usernames from `good_luck` and passwords from `Pass.txt`. 

When standard combinations failed, spraying the literal string `Pass.txt` as the password against the user `overflow` was successful:
`overflow` : `Pass.txt`
![](screenshots/Pasted%20image%2020260419122622.png)
![](screenshots/Pasted%20image%2020260419122806.png)

Initial access was established via SSH:
```bash
ssh overflow@192.168.29.80
whoami
overflow
```

## Privilege Escalation
### Local Kernel Enumeration
The operating system was identified as Ubuntu 14.04.

### Overlayfs Privilege Escalation
A local privilege escalation exploit (exploit 37292 / CVE-2015-1328) was compiled on the host and uploaded to the target machine:
![](screenshots/Pasted%20image%2020260419123801.png)

Executing the exploit spawned a root shell:
```bash
./exploit
whoami
root
```

## Post-Exploitation
### Flag Recovery
- Root Flag was successfully recovered from the root directory:
  ![](screenshots/Pasted%20image%2020260419123909.png)

## Lessons Learned
1. Disable anonymous FTP access and avoid leaving directories or sensitive files in plain packet captures.
2. Ensure system passwords are not named after files (e.g. `Pass.txt`) or easily guessable.
