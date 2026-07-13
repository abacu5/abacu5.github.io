---
title: "VulnHub: Simple"
platform: "VulnHub"
difficulty: "Easy"
os: "Linux (Ubuntu)"
excerpt: "A known CuteNews CMS file upload CVE for a foothold, then a public OverlayFS kernel exploit straight to root."
---

Simple is a beginner-friendly VulnHub machine built around a known CVE in the
CuteNews 2.0.3 CMS. The path is straightforward: exploit an unrestricted file
upload vulnerability via self-registration, get a reverse shell, then escalate to
root with a public OverlayFS kernel exploit. A good box for practicing the
"fingerprint the CMS version → search for a CVE → exploit" workflow that comes up
constantly in real-world engagements.

**Skills covered:**
- Network discovery with `netdiscover`
- Port & service enumeration with `nmap`
- CMS version fingerprinting
- CVE research and exploit identification (CuteNews 2.0.3 remote file upload)
- Unrestricted file upload via the avatar upload feature
- Reverse shell via a PHP webshell
- Linux kernel exploit  OverlayFS local root (CVE-2015-1328)

## Network Discovery

```
netdiscover -i eth0
```

Target resolved to `192.168.1.11`.

## Port Scanning

```
nmap -sV -sC 192.168.1.11
```

Only port 80 open, Apache 2.4.7 (Ubuntu). The scan immediately surfaced two useful
details: the HTTP title referenced CuteNews, and the CMS version  2.0.3  was
visible right in the page footer.

## Web Enumeration

The site was a CuteNews v2.0.3 login page. Running `dirb` turned up an `/uploads`
directory with directory listing enabled  meaning anything uploaded there would
be directly browsable. A strong early signal that file upload exploitation was
the intended path.

## CVE Research: CuteNews 2.0.3 Remote File Upload

Rather than spending time on credential guessing, checking the CMS version
against known vulnerabilities paid off immediately: CuteNews 2.0.3 has a
documented remote file upload vulnerability on Exploit-DB. The technique doesn't
need admin credentials at all  just a self-registered account:

1. Register a new account (registration was open)
2. Log in, go to Personal Options → avatar upload
3. Upload a PHP reverse shell disguised as an image
4. Intercept the request and rename the file extension from `.jpg` to `.php`
5. Access the shell at the resulting `/uploads/avatar_<username>_<filename>.php`
   path

## Exploiting the File Upload

Registered an account, then set up a PHP reverse shell with the attacker IP and
port configured. The application rejected `.php` uploads directly, so I
intercepted the request with Burp Suite and renamed the file in the
`Content-Disposition` header to end in `.php` instead of a safe image extension.
The upload succeeded, landing the shell in the public `/uploads/` directory.

## Getting the Reverse Shell

Started a listener:

```
nc -nvlp 7777
```

Navigated to the uploaded file's URL to trigger execution  got a connect-back
running as `www-data`.

## Kernel Version Fingerprinting

```
uname -a
```

Reported an Ubuntu 14.04 kernel from January 2015  old enough to be a strong
candidate for a known local privilege escalation exploit. Searching Exploit-DB for
that kernel range returned a solid hit: the OverlayFS local root exploit
(CVE-2015-1328), which abuses a flaw in the kernel's overlay filesystem
implementation to escalate to root without needing any sudo permissions or SUID
binaries.

## Privilege Escalation via OverlayFS Exploit

Hosted the exploit source on the attacker machine and pulled it onto the target
with `wget`, then compiled it directly there with `gcc` to avoid any
architecture mismatch between attacker and target:

```
gcc -o shell shell.c
./shell
```

The exploit ran through its mount sequence and dropped straight into a root
shell.

## Capturing the Flag

`/root/flag.txt` held the completion message, confirming full compromise.

## Attack Chain Summary

```
netdiscover → nmap (port 80: CuteNews 2.0.3)
  → dirb (/uploads, listing enabled)
  → CVE research → CuteNews remote file upload
  → self-register → avatar upload → renamed PHP shell → www-data
  → uname -a → old kernel → OverlayFS CVE-2015-1328
  → wget + gcc + ./shell → ROOT
```

## Key Takeaways

1. Always fingerprint the CMS version  CuteNews advertised its version right in
   the page footer. Version strings in footers, headers, and HTTP responses are
   one of the first things worth checking, since they map directly to known CVEs.
2. Open self-registration is a meaningful attack surface on its own. This
   vulnerability needed no admin credentials at all, just a valid registered
   account  any app allowing open self-registration needs strict controls on
   what authenticated users can actually do.
3. Unrestricted file upload remains a critical, common vulnerability class. The
   avatar feature accepted PHP files without server-side content-type
   validation  uploading executable code into a web-accessible directory is a
   direct line to remote code execution, and this still shows up regularly in
   real-world applications.
4. Kernel version matters for post-exploitation. Running `uname -a` immediately
   after landing a shell is a fundamental step  older, unpatched kernels are
   frequently vulnerable to public local privilege escalation exploits.
5. Compile exploits on the target when possible. Transferring source and
   compiling with `gcc` on-target avoids architecture mismatches  a pre-compiled
   binary can silently fail if the target is 32-bit and the attacker machine is
   64-bit.

## Tools Used

`netdiscover`, `nmap`, `dirb`, Exploit-DB, a PHP reverse shell, `netcat`, `wget`,
`gcc`.
