---
title: "VulnHub: CyberSploit 2"
platform: "VulnHub"
difficulty: "Easy"
os: "Linux (CentOS)"
excerpt: "ROT47-encoded credentials hiding on the homepage, then a Docker group misconfiguration straight to root."
---

CyberSploit 2 is a beginner-friendly VulnHub machine with a clever twist 
credentials hidden in plain sight using ROT47 encoding right on the web page
itself. Once inside, the privesc path runs through a Docker group
misconfiguration, a real-world vector that catches a surprising number of
production systems off guard too.

**Skills covered:**
- Network discovery with `netdiscover`
- Port & service enumeration with `nmap`
- Cipher recognition and decoding (ROT47)
- SSH login with discovered credentials
- Manual enumeration (`hint.txt`, `.bash_history`)
- Docker group privilege escalation via GTFOBins

## Network Discovery

```
netdiscover -i eth0
```

Target resolved to `192.168.1.6`.

## Port Scanning

```
nmap -T4 -A -p- 192.168.1.6
```

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 8.0 |
| 80/tcp | HTTP | Apache 2.4.37 (CentOS) |

The HTTP title read "CyberSploit2"  worth checking the page directly before
running any brute-force enumeration.

## Web Enumeration

The homepage displayed a table of usernames and passwords across several rows.
Most looked like plain, readable credentials  but one row stood out immediately:
both the username and password fields were clearly encoded, not natural text like
the rest of the table. A strong signal something was deliberately hidden there.

## Cipher Recognition and Decoding

Running the suspicious strings through a cipher identifier pointed to **ROT47** 
a substitution cipher that rotates printable ASCII characters by 47 positions.
Decoding both values recovered a working username and password pair.

Worth noting: ROT47 isn't encryption, just trivial obfuscation. Spotting
garbled-looking ASCII on a web page (that isn't hex or Base64) is a habit worth
building early in enumeration  ROT13/ROT47 is usually the first thing to try.

## SSH Login

Logged in with the recovered credentials over SSH, landing a shell as a
low-privileged user.

## Manual Enumeration

The home directory contained a `hint.txt` with a single word: **docker**.
Checking `.bash_history` backed that up  it was full of Docker commands (`docker
run`, `docker exec`, `docker image ls`, and more), confirming the escalation path
revolved around Docker.

Checking group membership:

```
id
```

The user belonged to the `docker` group. That's effectively equivalent to root on
the host, since Docker containers can mount the host filesystem. `sudo -l`
confirmed no sudo access  but none was needed.

## Privilege Escalation via Docker Group

Referencing GTFOBins' Docker entry, the standard group-escape technique mounts the
host's root filesystem into a container and drops into a shell inside it as root:

```
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

What each piece does:
- `-v /:/mnt`  mounts the entire host filesystem at `/mnt` inside the container
- `--rm`  removes the container after exit
- `-it alpine`  runs an interactive Alpine container
- `chroot /mnt sh`  changes root to the mounted host filesystem and drops into a
  shell

The container pulled Alpine and dropped straight into a root shell on the host.

## Capturing the Flag

`/root/flag.txt` held the completion message, confirming full compromise.

## Attack Chain Summary

```
netdiscover → nmap (SSH + HTTP) → homepage credential table
  → ROT47 decode → SSH login
  → hint.txt "docker" + .bash_history → id confirms docker group
  → GTFOBins docker escape (chroot host mount) → root
```

## Key Takeaways

1. Read every element of a web page carefully  the credentials were sitting on
   the homepage, just encoded well enough that a casual glance would miss them.
   Anything that looks "off" is worth a closer look.
2. ROT47 is a common CTF encoding trick. Garbled ASCII that isn't hex or Base64 is
   worth running through a cipher identifier before assuming it's a dead end.
3. The Docker group is a genuinely dangerous misconfiguration, not just a CTF
   trope. Adding a user to `docker` without understanding the implications grants
   effective root access to the host, no password required  worth auditing group
   memberships specifically for this during any privesc enumeration.
4. `.bash_history` is a goldmine. Commands run by previous users often reveal
   intended workflows, paths to escalation, or credentials left in plain text.
5. GTFOBins covers Docker directly  any time `id`, `groups`, or `sudo -l` shows
   Docker involvement, that's the first reference to check. The host filesystem
   mount technique works reliably across most Docker installs.

## Tools Used

`netdiscover`, `nmap`, a ROT47 cipher identifier, `ssh`, GTFOBins, `docker`.
