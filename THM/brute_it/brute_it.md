# TryHackMe — Brute It

**Platform:** TryHackMe
**Category:** Web / Brute Force / Privilege Escalation
**Difficulty:** Easy
**Date:** October 2026

---

## 1. Initial Reconnaissance

I started by scanning the target with Nmap to identify open ports and running services.

```bash
nmap -sC -sV 10.145.155.71
```

The scan revealed an HTTP service and SSH.

The HTTP service immediately became interesting because it gave me a web application to enumerate before attempting to access SSH.

---

## 2. Web Enumeration

I used Gobuster to search for hidden directories:

```bash
gobuster dir -u http://10.145.155.71 \
-w /usr/share/wordlists/dirb/common.txt
```

The interesting result was:

```text
/admin/    (Status: 301)
```

I opened:

```text
http://10.145.155.71/admin/
```

This revealed an administrator login page.

At this point, instead of blindly attacking SSH, I used Burp Suite to inspect how the login functionality worked and understand the application's requests and responses.

---

## 3. Discovering the SSH Private Key

While investigating the challenge, I eventually obtained an **encrypted RSA private key**.

The key itself was protected by a passphrase, so possessing the private key was not enough to authenticate.

I needed to recover its passphrase.

---

## 4. Converting the RSA Key for John

I used `ssh2john` to convert the encrypted SSH private key into a format that John the Ripper could process.

Conceptually:

```bash
ssh2john <private_key> > hash.txt
```

Then I used John:

```bash
john hash.txt
```

John successfully recovered the passphrase protecting the RSA private key.

I now had:

```text
Private RSA key
        +
Recovered passphrase
        ↓
SSH authentication
```

---

## 5. Getting an SSH Shell

I used the recovered private key to authenticate to the SSH service.

```bash
ssh -i <private_key> <username>@10.145.155.71
```

When SSH requested the passphrase, I entered the one recovered with John.

This gave me a shell on the target machine.

At this point, the attack changed from **remote enumeration** to **local enumeration and privilege escalation**.

---

# 6. Privilege Escalation Enumeration

Once inside the machine, I checked what commands I could execute with elevated privileges.

```bash
sudo -l
```

The output showed that `/bin/cat` could be executed with `sudo` without requiring my user's password.

This was the important privilege-escalation finding.

I had effectively found:

```text
User
 ↓
sudo permissions
 ↓
/bin/cat
 ↓
Read files as root
```

---

## 7. Reading `/etc/shadow`

Because `cat` could be executed through `sudo`, I tested whether I could use it to read sensitive files.

I ran:

```bash
sudo /bin/cat /etc/shadow
```

This worked.

The `/etc/shadow` file contains password hashes for local accounts, including the root account.

I copied the root password hash and saved it locally so I could attempt to crack it.

---

## 8. Cracking the Root Password

I used John the Ripper against the extracted hash.

```bash
john <hash_file>
```

John eventually recovered the plaintext root password.

At this point I had obtained the credentials necessary to become root.

---

# 9. Becoming Root

I switched to the root account:

```bash
su root
```

I entered the password recovered by John.

The shell changed to the root account.

I verified my privileges with:

```bash
whoami
```

The result confirmed:

```text
root
```

I had successfully obtained full control of the machine.

---


# What I Learned

### 1. Enumeration determines the attack path

The first useful discovery wasn't a password. It was:

```text
/admin/
```

That changed the direction of the investigation.

### 2. Burp is useful before automation

Instead of immediately throwing a huge wordlist at the target, I used Burp to understand the application's authentication behavior.

This is important because automation is much more effective when I understand the request I'm trying to automate.

### 3. An encrypted SSH key isn't useless

Finding a private key doesn't necessarily mean I can immediately use it.

If the key is encrypted, I may need to recover its passphrase first.

The chain:

```text
SSH key → ssh2john → John → passphrase
```

is a useful technique to remember.

### 4. `sudo -l` is one of the first commands to run after getting a shell

After gaining access to a Linux machine, I should check:

```bash
sudo -l
```

I am looking for commands that I can execute as another user, especially root.

### 5. `sudo` permissions can be more dangerous than they look

Being allowed to execute `/bin/cat` as root might initially seem harmless.

But `cat` can read arbitrary files.

That meant:

```bash
sudo /bin/cat /etc/shadow
```

allowed me to access password hashes that should normally be inaccessible to my user.

### 6. Privilege escalation can become another credential attack

The escalation wasn't a direct "run a root shell" situation.

Instead:

```text
sudo misconfiguration
       ↓
read /etc/shadow
       ↓
obtain root hash
       ↓
crack hash
       ↓
obtain root password
       ↓
su root
```

This was a good example of chaining multiple techniques together rather than relying on one exploit.

---

# Tools Used

| Tool            | Purpose                               |
| --------------- | ------------------------------------- |
| `nmap`          | Port and service enumeration          |
| `gobuster`      | Web directory enumeration             |
| Burp Suite      | HTTP request/response analysis        |
| `ssh2john`      | Convert SSH private-key data for John |
| John the Ripper | Password/passphrase cracking          |
| `ssh`           | Remote shell access                   |
| `sudo -l`       | Enumerate sudo privileges             |
| `cat`           | Read files                            |
| `su`            | Switch to another user                |

---
