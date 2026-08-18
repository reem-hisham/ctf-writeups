# Ubuntu 6 - SQL Injection Writeup

## Target

- **Machine:** Ubuntu 6
- **Target IP:** `192.168.58.133`
- **Main Vulnerability:** SQL Injection
- **Database:** MySQL

---

## 1. Enumeration

I started by discovering hosts on the network:

```bash
sudo arp-scan --localnet
````

Then I enumerated the web application and discovered several interesting paths:

```text
/admin/
/cat.php
/images/
/classes/
```

The `/admin/` directory redirected to `/admin/login.php`.

---

## 2. Finding SQL Injection

While testing the application, I found the following parameter:

```text
/cat.php?id=1
```

Adding unexpected input caused a MySQL syntax error:

```text
You have an error in your SQL syntax...
```

This confirmed that the `id` parameter was vulnerable to SQL Injection.

---

## 3. Database Enumeration

I used UNION-based SQL Injection to:

* Determine the number of columns.
* Identify columns that could display string values.
* Identify the current database.

The database name was:

```text
photoblog
```

Using `information_schema`, I then enumerated:

* Tables in the `photoblog` database.
* Columns in the interesting tables.
* Administrator credentials.

I used techniques such as `GROUP_CONCAT()` to retrieve multiple values through one displayed column and `CONCAT()` to combine values from multiple columns.

---

## 4. Admin Access

The extracted username was:

```text
admin
```

The password was stored as an MD5 hash.

After cracking the hash, I successfully authenticated to:

```text
/admin/login.php
```

The application used a `PHPSESSID` cookie to maintain the authenticated session.

---

## 5. Authenticated Enumeration

I continued enumeration using the authenticated session:

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

The admin panel allowed image uploads, and uploaded images were stored in this directory and displayed on the website.

---

## Attack Path

```text
ARP Scan
   ↓
Web Enumeration
   ↓
Discover /cat.php?id=
   ↓
MySQL Error
   ↓
Confirm SQL Injection
   ↓
UNION-based Enumeration
   ↓
Database: photoblog
   ↓
Tables & Columns Enumeration
   ↓
Extract Admin Credentials
   ↓
Crack MD5 Hash
   ↓
Admin Login
   ↓
Authenticated Enumeration
   ↓
Discover /admin/uploads/
   ↓
Test Image Upload
```

---

## Key Takeaways

* SQL errors can reveal the backend database technology.
* UNION-based SQL Injection can be used to enumerate databases, tables, and columns.
* `information_schema` is useful for MySQL database enumeration.
* `GROUP_CONCAT()` and `CONCAT()` help retrieve multiple values through limited output.
* Enumeration should continue after gaining authenticated access.
* Session cookies can be supplied to tools such as Gobuster for authenticated enumeration.
* File upload functionality increases the application's attack surface.

