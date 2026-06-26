# Offensive Security Certified Professional (OSCP) Prep Writeups

Welcome to my repository of walkthroughs and writeups for various cybersecurity target machines. This folder contains detailed walkthroughs of systems I have successfully compromised as part of my practical preparation before registering for the OffSec Certified Professional (OSCP) exam.

---

## Table of Contents

The repository contains walkthroughs for the following machines, hosted on VulnHub and OffSec Proving Grounds:

| Machine Name                                  | Platform        | Difficulty | Primary Focus / Techniques                                                                                                                 |
| --------------------------------------------- | --------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| [Amaterasu](Amaterasu.md)                     | Proving Grounds | Easy       | REST API Enumeration, SUID / Cronjob Tar Wildcard Injection                                                                                |
| [DC-9](DC-9.md)                               | Proving Grounds | Medium     | SQL Injection (UNION-based), Local File Inclusion (LFI), Port Knocking, Hydra SSH Brute Forcing, Priv Esc via Custom SUID/Writable Scripts |
| [DerpNStink](DerpNStink.md)                   | VulnHub         | Medium     | WordPress Slideshow Plugin RCE, PCAP Sniffing, Sudo Priv Esc                                                                               |
| [Gaara](Gaara.md)                             | Proving Grounds | Easy       | Directory Fuzzing, Base58 Decryption, SUID GDB Exploitation                                                                                |
| [GoldenEye v1](GoldenEye%20v1.md)             | VulnHub         | Medium     | POP3/SMTP Credential Enumeration, Moodle Spellchecker RCE, Kernel Exploit                                                                  |
| [LazySysAdmin](LazySysAdmin.md)               | VulnHub         | Easy       | SMB Guest Access, WordPress Config Leak, SSH Password Spraying                                                                             |
| [Lord of the Root](Lord%20of%20the%20Root.md) | VulnHub         | Medium     | Port Knocking, SQL Injection (Sqlmap), SSH Hydra Spray, Kernel Exploit                                                                     |
| [SickOS](SickOS.md)                           | VulnHub         | Medium     | Squid Proxy Routing, WolfCMS Shell Upload, Writable Python Cronjob                                                                         |
| [Stapler](Stapler.md)                         | VulnHub         | Medium     | FTP Anonymous Login, PCAP Analysis, Hydra SSH Brute Forcing, Writable Cronjob Privilege Escalation                                         |
| [The Ether](The%20Ether.md)                   | VulnHub         | Medium     | LFI Auth Log Poisoning, Sudo Script Command Injection, Steganography                                                                       |
| [Troll](Troll.md)                             | VulnHub         | Easy       | FTP Anonymous Login, PCAP Extraction, Password Spraying, Kernel Exploit                                                                    |
| [Wintermute](Wintermute.md)                   | VulnHub         | Hard       | Double Subnet Pivoting, SMTP Shellshock, Struts2 RCE, Socat Redirects                                                                      |

---

## How to Use These Writeups

These writeups are structured to mirror the standard OSCP report format as closely as possible:

1. **Overview**: Key metadata regarding the machine (OS, IP, platform, difficulty, and focus areas).
2. **Enumeration**: A step-by-step breakdown of network discovery, port scanning, and service-specific enumeration (Web, SMB, database, etc.).
3. **Exploitation**: Detailed instructions on obtaining initial access, including scripts used, listener settings, and code execution parameters.
4. **Privilege Escalation**: Steps taken to perform vertical/lateral user movement and secure root-level permissions.
5. **Post-Exploitation**: Details on flag locations and recovery proof.
6. **Lessons Learned**: Tactical takeaways identifying the configuration/security failures on the machine and how to defend against them.

*Note: All screenshots referenced within the markdown notes are stored in the local `./screenshots/` directory.*

---

## Commonly Used Tools

Throughout these machines, the following security tools and utilities were heavily utilized:

- **Reconnaissance & Enumeration**: `Nmap`, `arp-scan`, `Nikto`, `ffuf`, `Dirbuster`
- **Exploitation & Scanning**: `Sqlmap`, `Hydra`, `Metasploit (msfconsole)`, `Socat`
- **Post-Exploitation & Privilege Escalation**: `LinPEAS`, `GTFOBins`, `dos2unix`, static GCC compilation toolchains, and custom script hijacking.

---

## Disclaimer

These writeups are created strictly for educational purposes, personal study, and preparation for the OSCP certification exam. All testing was performed in a isolated, self-hosted local laboratory environment or on authorized training platforms (OffSec Proving Grounds). Unauthorized scanning, exploitation, or attacking of systems without explicit prior permission is illegal.
