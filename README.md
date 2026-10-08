# SQL Injection Attack & Defense Lab

A hands-on security lab demonstrating SQL injection vulnerabilities in a PHP/MySQL web application — from exploitation through remediation — built on SEED Labs' containerized Employee Management System.

## Overview

This project explores SQL injection (SQLi) as both an attacker and a defender. Using a deliberately vulnerable employee management web app, I demonstrated authentication bypass, data exfiltration, and privilege escalation attacks, then fixed the underlying vulnerability using parameterized prepared statements — verifying the fix against the same payloads that previously succeeded.

**Lab source:** [SEED Labs — SQL Injection Attack Lab](https://seedsecuritylabs.org/Labs_20.04/Web/Web_SQL_Injection/)

## Environment Setup

- **Host:** VMware Fusion VM running Ubuntu 22.04 LTS (ARM64 / Apple Silicon)
- **Containerization:** Docker Compose, two containers:
  - `www` — Apache + PHP web application (`10.9.0.5`)
  - `mysql` — MySQL 8.0 database server (`10.9.0.6`)
- **Networking:** Custom Docker bridge network; `/etc/hosts` mapping for `www.seed-server.com`
- **Notable setup challenge:** Resolved an `aarch64`/ARM image-compatibility issue (standard `Labsetup.zip` ships x86_64-only MySQL images) by using the ARM-specific `Labsetup-arm.zip`, and extended the VM's LVM logical volume to resolve disk space exhaustion during MySQL initialization.

## Vulnerability

The application's `unsafe_home.php` and `unsafe_edit_backend.php` endpoints build SQL queries via direct string concatenation of user input:

```php
$sql = "SELECT id, name, eid, salary, birth, ssn, address, email, nickname, Password
        FROM credential
        WHERE name='$input_uname' and Password='$hashed_pwd'";
$result = $conn->query($sql);
```

Because `$input_uname` is never sanitized or parameterized, attacker-controlled input is interpreted as part of the SQL syntax rather than as data.

## Attacks Demonstrated

### 1. Authentication Bypass (Task 2.1 / 2.2)
Injected `admin' #` as the username, commenting out the password check entirely:
```bash
curl "www.seed-server.com/unsafe_home.php?username=admin%27%20%23&Password=x"
```
**Result:** Authenticated as `admin` with no valid password, returning the full employee table (all names, salaries, SSNs).

### 2. Stacked Query Attempt (Task 2.3)
Attempted to append a second SQL statement via semicolon:
```bash
curl "www.seed-server.com/unsafe_home.php?username=admin%27%3B%20SELECT%201%3B%20%23&Password=x"
```
**Result:** Blocked — PHP's `mysqli::query()` only executes a single statement per call. (Incidental protection, not a deliberate security control; `mysqli_multi_query()` would have allowed it.)

### 3. Privilege Escalation via UPDATE Injection (Task 3.1–3.3)
Logged in as a low-privilege user (`Alice`) and exploited the Edit Profile endpoint's unsanitized `NickName` field to:
- **Modify own salary** — injected an extra `SET` clause (`salary='1000000'`)
- **Modify another user's salary** — injected a second `WHERE` clause to retarget the UPDATE at `Boby`'s row, bypassing the session-bound `WHERE ID=$id` restriction
- **Full account takeover** — injected a precomputed SHA1 hash directly into another user's `Password` column, then logged in as that user with the corresponding plaintext password

Example payload (account takeover):
