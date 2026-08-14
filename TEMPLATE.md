# HackTheBox — <BOX NAME>

> **OS:** <Linux/Windows>  ·  **Difficulty:** <Easy/Medium/Hard>  ·  **Date:** <YYYY-MM-DD>

**Kill chain:** `<one-line summary of the full path from recon to root>`

---

## Overview

<2–4 sentences: what the box is, the main vuln(s), and the shape of the privesc.
Call out anything unusual about the environment — defensive controls, weird config,
rabbit holes worth warning about.>

---

## Recon

### Port scan

```bash
nmap -sV -sC <ip>
```

```
<paste relevant open ports + services>
```

<What stood out and why. What you enumerated next.>

### Service / web enumeration

```bash
<commands>
```

<Findings — usernames, versions, hostnames, disclosed files. Fingerprint the
*application*, not just the server.>

---

## Exploitation

<Name the CVE / technique / EDB ID. Explain *why* it applies (version match, etc).
If you had to adapt tooling — throttling, porting, custom payload — document that,
it's the most valuable part of the report.>

```bash
<exploit command>
```

```
<result — credential, shell, etc>
```

---

## Foothold

<How the exploit turned into access. Credential reuse? Upload? Deserialization?>

```bash
<command>
```

```
<user.txt: XXXXXXXX…[REDACTED]>
```

---

## Privilege Escalation

<Enumeration approach (linpeas/pspy/manual). The chain of facts that made privesc
possible. Show the trigger, the primitive, and the payload.>

```bash
<commands>
```

```
<root.txt: XXXXXXXX…[REDACTED]>
```

---

## Remediation

| Finding | Severity | Fix |
|---------|----------|-----|
| <vuln> | <Critical/High/Medium/Low> | <concrete remediation> |

---

## Detection notes (blue-team view)

<What a defender would see in logs / telemetry for each stage. Which events are
high-signal. What detection or control would break the chain.>

---

## Lessons

- <takeaway>
- <takeaway>
