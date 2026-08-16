# Penetration Test Reports — HackTheBox

Retired HackTheBox machines, written up the way you'd write a real engagement.

Not CTF walkthroughs. Every report has the same three things a walkthrough skips:

- The reasoning behind each step, not just the commands.
- A findings table with severities and fixes.
- A detection section: what the attack looks like in logs, and how a defender catches it.

I do detection engineering for a living. So for every box I break, I write down how I'd catch someone doing the same thing. That's the part I care about most.

> Retired machines only. No active-box solutions.

---

## Reports

| # | Machine | OS | Difficulty | Way in | Root | Report |
|---|---------|----|-----------|--------|------|--------|
| 1 | **Writeup** | Linux | Easy | CMS Made Simple SQLi (CVE-2019-9053) | PATH hijack | [→](writeup/README.md) |
| 2 | **Blue** | Windows | Easy | SMB, MS17-010 / EternalBlue | RCE straight to SYSTEM | [→](blue/README.md) |
| 3 | **Knife** | Linux | Easy | PHP 8.1.0-dev backdoor RCE | sudo on knife (runs Ruby as root) | [→](knife/README.md) |
| 4 | **Keeper** | Linux | Easy | Request Tracker default creds | KeePass CVE-2023-32784 -> root's key | [→](keeper/README.md) |
| 5 | **Wifinetic** | Linux | Easy | Anonymous FTP config leak | cap_net_raw on reaver -> WPS attack | [→](wifinetic/README.md) |

---

## What these cover

Not a box count. What the reports actually show I can do.

**Getting in**
SQL injection, known-CVE exploitation, default and reused credentials, FTP enumeration, backdoored-package RCE, SMB / MS17-010.

**Privilege escalation**
PATH hijacking, sudo misconfig on code-exec binaries, Linux capabilities, credential reuse across services, unpatched SMB to SYSTEM.

**After the foothold**
Hash cracking with john and hashcat, KeePass memory-dump recovery, PuTTY to OpenSSH key conversion, wireless WPS attacks with reaver.

**Tradecraft I write down**
Shell stabilisation and PTY upgrades, working around broken shells that mangle quotes and redirects, enumerating off evidence instead of guessing, and detection notes on every finding.

---

## How each report is laid out

Same template every time (`TEMPLATE.md`):

Overview, recon, exploitation, privilege escalation, remediation, detection notes, lessons.

---

## About

Me: [Mysteryneon](https://github.com/Mysteryneon), OSCP. I'm doing these to get sharp at reporting, which is the thing most people skip and most jobs actually need.

Next: medium boxes, and more Active Directory.
