---
title: "HTB: Nibbles"
platform: "Hack The Box (retired)"
difficulty: "Easy"
os: "Linux"
excerpt: "A hidden CMS path, a decade-old Nibbleblog file upload CVE, and a writable script with passwordless sudo for root."
---

Nibbles is a classic beginner-friendly HTB machine that chains together three core
skills: web enumeration, exploiting a known CVE in a blogging CMS, and abusing a
misconfigured sudo rule for privilege escalation. Full path from initial scan to
root, using Nmap, ffuf/Gobuster, and manual privesc  no automated exploitation
frameworks required.

## Reconnaissance

Started with a full TCP port scan:

```
nmap -sC -sV -p- <target>
```

Only two ports open: 22/tcp (OpenSSH 7.2p2 Ubuntu) and 80/tcp (Apache 2.4.18
Ubuntu). No SSH creds yet, so port 80 was the obvious starting point. The HTTP
title came back blank  often a sign the default Apache landing page has been
swapped out for something custom.

## Web Enumeration

A directory brute-force on the web root came back empty aside from default Apache
protection files:

```
ffuf -u http://<target>/FUZZ -w /usr/share/wordlists/dirb/big.txt -t 100
```

Checking the raw page source, though, revealed an HTML comment pointing to a
hidden directory. Fuzzing inside that path  this time with extensions, since PHP
apps won't surface otherwise  confirmed a CMS install:

```
ffuf -u http://<target>/nibbleblog/FUZZ -w /usr/share/wordlists/dirb/common.txt -e .php,.html,.txt,.bak -t 100
```

**Lesson:** Gobuster and ffuf only test raw wordlist entries by default  they
won't guess file extensions unless you explicitly pass `-x` (gobuster) or `-e`
(ffuf). This is a common reason legitimate files like `admin.php` get missed on a
first pass.

Browsing confirmed the CMS was Nibbleblog, with an admin panel at
`/nibbleblog/admin.php`.

## Identifying the Vulnerability

Nibbleblog exposes some of its configuration in an XML file readable without
authentication:

```
curl -s http://<target>/nibbleblog/content/private/users.xml
```

This revealed the admin username and other account metadata. Given the CMS and
version, the relevant vulnerability is **CVE-2015-6967**  an authenticated
arbitrary file upload flaw in Nibbleblog's "My Image" plugin. An authenticated
user can upload a malicious `.php` file disguised as an image, achieving remote
code execution once the file is accessed directly.

**Finding admin credentials:** obvious guesses like `admin:admin` didn't work, and
the app locks out an IP after a handful of failed attempts, ruling out a
straightforward wordlist attack on the login form. What worked instead: the CMS's
own branding  site title, notification email, and the machine name itself  leaned
heavily on the word "nibbles." Trying the recovered username against that word as
the password logged straight in.

This is a good reminder that CMS installs often leak their own password
indirectly through branding, titles, or config files exposed via directory
listing  worth checking before brute-forcing.

## Exploitation  Getting a Shell

1. Logged into the admin panel with the recovered credentials.
2. Went to Plugins → My Image → Configure. This plugin accepts an image for the
   site header but doesn't properly validate the file extension.
3. Uploaded a PHP web shell disguised as an image. The plugin throws
   image-processing errors after the upload (expected, since it isn't a valid
   image)  but the file saves anyway.
4. The uploaded file landed in a predictable path under the plugin's content
   directory.
5. Confirmed code execution with a test payload, then swapped in a reverse shell
   one-liner using `mkfifo` and `nc`.
6. Started a listener, re-uploaded the modified file, and browsed to it to
   trigger execution.

This landed a shell as the low-privileged web user. (Note: a Metasploit module 
`exploit/multi/http/nibbleblog_file_upload`  automates this entire exploit chain
if manual isn't required.)

## Privilege Escalation

Basic home directory enumeration revealed a personal archive. Unzipping it turned
up a script, owned by the low-privileged user and world-writable.

Checking sudo permissions confirmed the escalation path:

```
sudo -l
```

The low-privileged user could run that same script as root, no password required
 and also had write access to it. Classic combination: overwrite the script,
then trigger it via sudo to get a root shell.

```
printf '#!/bin/bash\n/bin/bash -p\n' > <path-to-script>
chmod +x <path-to-script>
sudo <path-to-script>
```

`id` confirmed root. Grabbed `/root/root.txt` to close it out.

## Key Takeaways

1. Directory brute-forcing needs extensions. Both Gobuster (`-x`) and ffuf (`-e`)
   require explicitly specifying file extensions, or common files like
   `admin.php` get silently skipped.
2. Check page source before assuming a dead end. A blank Apache title didn't mean
   nothing was there  the real application path was hidden in an HTML comment.
3. Known CVEs on older CMS versions are still some of the easiest wins.
   CVE-2015-6967 is over a decade old but remains a reliable authenticated RCE
   path wherever patching has been neglected.
4. Writable scripts plus passwordless sudo is a classic privesc combo. Always run
   `sudo -l` early in enumeration, and cross-reference it against any writable
   files already found  the intersection is often the fastest path to root.
