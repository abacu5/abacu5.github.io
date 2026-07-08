---
title: "VulnHub: C0ldBox"
platform: "VulnHub"
difficulty: "Easy"
os: "Linux (Ubuntu)"
excerpt: "WordPress theme editor RCE, credential reuse from wp-config.php, and sudo/ftp privesc via GTFOBins."
---

C0ldBox is a beginner-friendly VulnHub machine that walks through a classic attack
chain: network discovery, WordPress enumeration, brute-force login, reverse shell via
theme editor, credential reuse for lateral movement, and privilege escalation via a
sudo misconfiguration. A solid box for anyone starting out with CTFs or pentest
fundamentals.

**Skills covered:**
- Network discovery with `netdiscover`
- Port/service scanning with `nmap`
- Directory brute-forcing with `dirb`
- WordPress credential attack with `wpscan`
- Reverse shell injection via WordPress theme editor
- Lateral movement using credentials from `wp-config.php`
- Privilege escalation via GTFOBins (`ftp` sudo exploit)

## Network Discovery

Found the target on the local subnet with an ARP scan:

```
netdiscover -i eth0
```

The target resolved to `192.168.1.3` — a VirtualBox adapter MAC, confirming the VM.

## Port Scanning

```
nmap -sV -sC 192.168.1.3
```

| Port | Service | Version |
|------|---------|---------|
| 80/tcp | HTTP | Apache 2.4.18 (Ubuntu), WordPress 4.1.31 |
| 4512/tcp | SSH | OpenSSH 7.2p2 Ubuntu |

Worth noting: SSH on a non-standard port (4512) — potentially useful later for lateral
movement if credentials turn up.

## Web Enumeration

The site at `http://192.168.1.3` was a minimal WordPress homepage with no obvious
leads. Ran `dirb` to brute-force directories:

```
dirb http://192.168.1.3
```

Turned up `/wp-admin`, `/wp-content`, and a hidden `/hidden/` directory. The
`/wp-admin` login was live but default credentials didn't work.

## WordPress Brute Force

Guessed a likely username from the box name (`c0ldd`) and ran `wpscan` against it
with rockyou.txt:

```
wpscan --url http://192.168.1.3 --usernames c0ldd --passwords /usr/share/wordlists/rockyou.txt
```

Got a hit after roughly 1,300 requests, giving valid WordPress admin credentials.

## RCE via Theme Editor

With dashboard access, I went to **Appearance → Editor → header.php** (Twenty
Fifteen theme) and replaced its contents with a Pentestmonkey PHP reverse shell,
pointed at my attacker IP and a listener port.

Set up the listener:

```
nc -nlvp 7777
```

Refreshing the WordPress homepage triggered the shell — an immediate connect-back as
`www-data`. Upgraded it to a proper TTY:

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

## Lateral Movement via wp-config.php

WordPress config files routinely hold database credentials, so I checked:

```
cat /var/www/html/wp-config.php
```

That surfaced a DB username/password pair. Since credential reuse across
application and system accounts is common, I tried switching to the matching
system user with the same password — and it worked, landing a shell as `c0ldd`.

## Privilege Escalation via Sudo + GTFOBins

```
sudo -l
```

`c0ldd` had passwordless sudo rights on three binaries: `vim`, `chmod`, and `ftp`.
All three have known GTFOBins privesc paths — I went with `ftp`:

```
sudo ftp
ftp> !/bin/sh
```

That dropped into a root shell, confirmed with `id` and `whoami`. Grabbed `root.txt`
from `/root` to close it out.

## Attack Chain Summary

```
netdiscover → nmap (port 80: WordPress) → dirb (/wp-admin)
  → wpscan brute force (c0ldd) → theme editor RCE → www-data shell
  → wp-config.php creds → su c0ldd → sudo -l → ftp → root
```

## Key Takeaways

1. Non-standard service ports (like SSH on 4512) are worth flagging early — they
   can become useful once credentials are in hand.
2. WordPress theme/plugin editors are a classic, reliable RCE vector once admin
   access is obtained.
3. Config files like `wp-config.php` frequently leak credentials that get reused
   at the system level — always check them.
4. GTFOBins is essential reading. Any time `sudo -l` shows binaries like `vim`,
   `ftp`, or `chmod`, there's almost always a privesc path.
5. Sudo misconfigurations remain one of the most common real-world privesc
   vectors, not just a CTF trope.

## Tools Used

`netdiscover`, `nmap`, `dirb`, `wpscan`, Pentestmonkey PHP reverse shell,
`netcat`, GTFOBins.
