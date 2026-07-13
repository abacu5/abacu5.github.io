---
title: "VulnHub: OSCP"
platform: "VulnHub"
difficulty: "Easy"
os: "Linux (Ubuntu 20.04)"
excerpt: "A username hiding in plain sight, a Base64-encoded SSH private key, and instant root via a SUID bash binary."
---

The OSCP box on VulnHub is a clean, beginner-friendly machine simulating a
realistic attack path without unnecessary complexity. The key techniques are
reading between the lines on a web page, recognizing a Base64-encoded SSH private
key hiding in plain sight, and exploiting a SUID bash binary for instant root.
Short chain, but every step teaches something worth keeping.

**Skills covered:**
- Network discovery with `netdiscover`
- Port & service enumeration with `nmap`
- Web content enumeration (`robots.txt`, `secret.txt`)
- Base64 decoding of an SSH private key
- SSH login using a private key
- SUID bit exploitation (`/usr/bin/bash -p`)

## Network Discovery

```
netdiscover -i eth0
```

Target resolved to `192.168.1.8`.

## Port Scanning

```
nmap -T4 -A -p- 192.168.1.8
```

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 8.2p1 (Ubuntu) |
| 80/tcp | HTTP | Apache 2.4.41, WordPress 5.4.2 |
| 33060/tcp | mysqlx | MySQL X Protocol |

The scan also flagged a `robots.txt` disallowing a single path — `/secret.txt` —
worth checking immediately.

## Web Enumeration

The WordPress homepage read normally at a glance, but the post content itself
included a clear hint near the bottom: the only user on the box was named
explicitly. That's a username worth noting straight away, no fuzzing required.

`robots.txt` confirmed the single disallowed entry pointing to `/secret.txt`,
which loaded as a wall of Base64-encoded text.

## Decoding the SSH Private Key

Pulled and decoded the file directly:

```
curl http://192.168.1.8/secret.txt | base64 -d > id
```

The decoded output was a complete OpenSSH private key, ready to use for
authentication. Set the correct permissions before SSH would accept it:

```
chmod 600 id
```

## SSH Login with Private Key

```
ssh oscp@192.168.1.8 -i id
```

Logged straight in — no password needed, the key handled authentication.

## Privilege Escalation via SUID bash

Searched for SUID binaries:

```
find / -perm -u=s -type f 2>/dev/null
```

One entry stood out immediately: `/usr/bin/bash` had the SUID bit set, meaning it
runs with the file owner's privileges (root) regardless of who executes it.
Running it with the `-p` flag, which preserves the effective UID:

```
cd /usr/bin
./bash -p
```

`id` confirmed an effective UID of 0 — root.

## Capturing the Flag

`/root/flag.txt` held the completion flag, confirming full compromise.

## Attack Chain Summary

```
netdiscover → nmap (SSH + WordPress)
  → page content reveals username
  → robots.txt → /secret.txt → Base64 decode → SSH private key
  → ssh -i id → shell as user
  → find SUID binaries → /usr/bin/bash
  → ./bash -p → euid=0 (root)
```

## Key Takeaways

1. Read web page content thoroughly before reaching for tools. The username was
   stated explicitly in the post body — skimming straight to directory fuzzing
   would have missed it entirely.
2. `robots.txt` disallowed entries are explicit hints in a CTF context — always
   check every disallowed path rather than assuming it's a dead end.
3. Base64 can encode entire files, not just passwords. An SSH private key is just
   text, and text encodes cleanly. Any large Base64 block is worth decoding and
   checking the file type on, not dismissing as "probably just a credential."
4. SSH private key permissions matter — `chmod 600` isn't optional. SSH will
   refuse a key with overly permissive file permissions as a security measure,
   which trips up a lot of beginners the first time they hit it.
5. SUID on `/bin/bash` or `/usr/bin/bash` is close to an instant root. The `-p`
   flag tells bash to start in privileged mode, preserving the effective UID set
   by the SUID bit — one of the most well-known Linux privesc misconfigurations,
   worth checking for on every box.

## Tools Used

`netdiscover`, `nmap`, `curl`, `base64`, `ssh`, `find`.
