# Debian 6 — SQL Injection Writeup

## Target

* **Machine:** Debian 6
* **Target IP:** `192.168.58.133`
* **Main Vulnerability:** SQL Injection
* **Database:** MySQL

---

## 1. Objective

The objective was to compromise the Debian 6 target by identifying vulnerabilities in the exposed web application and gaining administrative access.

The main attack path involved exploiting a SQL Injection vulnerability to retrieve administrator credentials, authenticate to the admin panel, and continue enumerating the application.

---

## 2. Reconnaissance

I started by discovering active hosts on the local network:

```bash
sudo arp-scan --localnet
```

The target was identified as:

```text
192.168.58.133
```

I then enumerated the web application and discovered several interesting paths:

```text
/admin/
/cat.php
/images/
/classes/
```

The `/admin/` directory redirected to:

```text
/admin/login.php
```

---

## 3. Finding SQL Injection

While testing the application, I discovered the following parameter:

```text
/cat.php?id=1
```

I modified the `id` parameter with unexpected SQL syntax, which caused the application to return a MySQL error:

```text
You have an error in your SQL syntax...
```

This indicated that user input was being directly incorporated into a SQL query and suggested that the `id` parameter was vulnerable to SQL Injection.

---

## 4. SQL Injection Analysis & Database Enumeration

I tested the parameter further and confirmed that the vulnerability could be exploited using a **UNION-based SQL Injection**.

The first step was determining the number of columns returned by the original query. I then identified which columns could display string data.

After confirming the query structure, I used the injection point to enumerate database information.

The current database was identified as:

```text
photoblog
```

I then queried MySQL's `information_schema` to enumerate:

* Available tables
* Columns within interesting tables
* Administrator-related data

To make the extraction more efficient, I used functions such as:

```sql
GROUP_CONCAT()
```

to retrieve multiple values through a single output column, and:

```sql
CONCAT()
```

to combine multiple fields into one result.

This ultimately allowed me to retrieve the administrator credentials.

---

## 5. Credential Recovery

The administrator username was:

```text
admin
```

The password was stored as an **MD5 hash**.

After cracking the hash, I was able to authenticate successfully through:

```text
/admin/login.php
```

The application created a `PHPSESSID` cookie to maintain the authenticated session.

---

## 6. Authenticated Enumeration

After obtaining authenticated access, I continued enumerating the application rather than stopping at the login page.

I used Gobuster while supplying the authenticated session:

```bash
gobuster dir \
-u http://192.168.58.133/admin/ \
-w /usr/share/wordlists/dirb/common.txt \
-c "PHPSESSID=<SESSION_ID>"
```

This revealed an interesting directory:

```text
/admin/uploads/
```

Further investigation showed that the admin panel provided image-upload functionality and that uploaded files were stored in this directory and displayed by the website.

This identified a second potentially interesting attack surface: **file upload functionality**.

---

## 7. Attack Path

```text
ARP Scan
   ↓
Web Enumeration
   ↓
Discover /cat.php?id=
   ↓
Trigger MySQL Error
   ↓
Confirm SQL Injection
   ↓
Determine UNION Query Structure
   ↓
Identify Database: photoblog
   ↓
Enumerate Tables & Columns
   ↓
Extract Administrator Credentials
   ↓
Crack MD5 Password
   ↓
Authenticate to Admin Panel
   ↓
Authenticated Directory Enumeration
   ↓
Discover /admin/uploads/
   ↓
Identify File Upload Functionality
```

---

## 8. Lessons Learned

* SQL error messages can reveal the backend database technology.
* UNION-based SQL Injection can be used to extract information from a vulnerable query.
* MySQL's `information_schema` is useful for enumerating databases, tables, and columns.
* `GROUP_CONCAT()` can help extract multiple database values through a limited number of output columns.
* `CONCAT()` is useful for combining multiple fields into a single result.
* Password hashes obtained through SQL Injection may allow further access if they can be cracked.
* Enumeration should continue after obtaining authenticated access.
* Session cookies can be supplied to tools such as Gobuster for authenticated enumeration.
* File upload functionality should always be examined carefully because it can introduce additional attack paths.

---

## Conclusion

The compromise began with a SQL Injection vulnerability in the `id` parameter of `/cat.php`. By analyzing the SQL query and using UNION-based injection, I was able to enumerate the `photoblog` database and retrieve administrator credentials.

After cracking the MD5 password, I gained access to the admin panel and continued enumeration, discovering an authenticated upload directory.

The main lesson from this machine was the importance of **methodical enumeration and following the attack chain step by step**, rather than stopping immediately after discovering the first vulnerability.
