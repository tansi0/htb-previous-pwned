# HTB — Previous (Linux, Medium)

**Platform:** Hack The Box  
**OS:** Linux  
**Difficulty:** Medium  
**Release Date:** August 23, 2025  
**Completed:** November 2025  
**Status:** Pwned ✅ (User + Root flags obtained)

---

## Summary

Previous is a medium-difficulty Linux machine built around a Next.js web application with misconfigured API middleware. The attack chain involves web enumeration, Local File Inclusion via directory traversal, credential extraction from server-side source code, SSH access as a low-privileged user, and privilege escalation through a Terraform provider hijack.

## Attack Path

```
Nmap scan → Dirsearch (API endpoint discovery) → FFUF parameter fuzzing
→ LFI via /api/download?example=../../../../etc/passwd
→ Next.js build artifact extraction → Credential recovery (jeremy)
→ SSH login → User flag
→ Terraform provider hijack (chmod u+s /bin/bash) → Root shell → Root flag
```

## Key Techniques

- Directory traversal / Local File Inclusion (LFI)
- Next.js server-side source code extraction
- Credential reuse across web app and SSH
- Terraform provider binary override for privilege escalation
- SUID bash abuse to spawn root shell

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service discovery |
| Dirsearch | API endpoint enumeration, middleware bypass |
| FFUF | Parameter fuzzing on download endpoint |
| curl | LFI exploitation, file retrieval via path traversal |
| jq | JSON parsing of Next.js build artifacts |
| SSH | Remote access using recovered credentials |
| Terraform CLI | Provider hijack privilege escalation |
| Bash | One-liners for file inspection and archive processing |

## Vulnerability Summary

| # | Finding | Severity | Impact |
|---|---------|----------|--------|
| 1 | Local File Inclusion via API parameter | High | Arbitrary file read on server filesystem |
| 2 | Credentials exposed in Next.js build artifacts | Critical | Direct SSH access as system user |
| 3 | Terraform provider path misconfiguration | High | Full root privilege escalation |
| 4 | Credential reuse (web → SSH) | Medium | Lateral movement without additional exploitation |

## Writeup

See [writeup.md](writeup.md) for the full step-by-step walkthrough.

---


