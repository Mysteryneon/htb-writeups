# HTB Write-ups

Penetration testing write-ups for retired [HackTheBox](https://www.hackthebox.com/) machines.
One box per folder, each with full recon → foothold → privesc → remediation, plus a blue-team
detection view.

> Only **retired** boxes are published here. Active-machine solutions are kept private per HTB policy.

## Index

| Box | OS | Difficulty | Key techniques | Write-up |
|-----|----|-----------|----------------|----------|
| Writeup | Linux | Easy | CVE-2019-9053 (CMSMS SQLi), password reuse, PATH hijack | [→](writeup/README.md) |
| Blue | Windows | Easy | MS17-010 / EternalBlue, unauthenticated RCE as SYSTEM | [→](blue/README.md) |
| Knife | Linux | Easy | PHP 8.1.0-dev backdoor RCE, sudo knife exec privesc | [→](knife/README.md) |
| Keeper | Linux | Easy | RT default creds, KeePass CVE-2023-32784, root PuTTY key | [→](keeper/README.md) |
| Wifinetic | Linux | Easy | Anon FTP config leak, PSK reuse, reaver cap_net_raw WPS attack | [→](wifinetic/README.md) |

## Structure

```
htb-writeups/
├── README.md          ← this index
├── TEMPLATE.md        ← copy this to start a new box
├── PUBLISHING.md      ← how to render these as a public site
├── .gitignore         ← keeps sensitive/private notes out of the repo
└── <box-name>/
    ├── README.md      ← the write-up
    └── assets/        ← screenshots, exploit scripts
```

## Adding a new box

```bash
cp TEMPLATE.md <box-name>/README.md
mkdir -p <box-name>/assets
# write it up, drop screenshots in assets/, then add a row to the Index table above
```

## About

Maintained as deliberate reporting practice — the goal is clear, reproducible,
professional-grade documentation, not just flags. Feedback welcome.
