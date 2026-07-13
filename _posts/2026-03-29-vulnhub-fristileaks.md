---
title: "VulnHub: FristiLeaks 1.3"
platform: "VulnHub"
difficulty: "Easy-Medium"
os: "Linux (CentOS)"
excerpt: "Robots.txt rabbit holes, a password hidden in a Base64-encoded image, a file upload bypass, and a custom encoder reversed for root."
---

FristiLeaks 1.3 is a layered VulnHub machine that rewards patience and lateral
thinking over just running tools. It's packed with deliberate misdirection: a
`robots.txt` full of rabbit holes, credentials hidden inside a Base64-encoded image
in the page source, a file upload filter that needs bypassing, and a multi-stage
privesc chain running through cron jobs, a custom encoding script, and a hidden
sudo binary.

**Skills covered:**
- Network discovery with `netdiscover`
- Port & service enumeration with `nmap`
- `robots.txt` enumeration and rabbit hole identification
- Page source analysis and Base64 image decoding
- File upload filter bypass (extension spoofing)
- Reverse shell via a PHP webshell
- Cron job abuse for lateral movement
- Reverse engineering a custom Python encoder
- Multi-hop privesc via a custom sudo binary

## Network Discovery

```
netdiscover -i eth0
```

Target resolved to `192.168.1.17`.

## Port Scanning

```
nmap -T4 -A -p- 192.168.1.17
```

Only one port open: 80/tcp, Apache 2.2.15 (CentOS) with DAV/2 and PHP/5.3.3. The
scan also flagged a `robots.txt` with three disallowed entries, and an unusual
"Site doesn't have a title" HTTP title — both worth digging into.

## Web Enumeration

The homepage was a themed page with nothing actionable in its source.
`robots.txt` disallowed three paths — `/cola`, `/sisi`, `/beer` — and each one
just served a troll image. Classic rabbit hole.

The pattern was drink-themed, and the homepage prominently referenced "fristi," so
guessing a hidden `/fristi` path paid off — it revealed an admin login portal.

## Credentials from Page Source

Default credentials and basic SQLi attempts both failed, so it was time to read
the page source carefully. Two things stood out in the HTML:

- A developer comment leaking a username directly
- A large Base64 string embedded as an image source

Decoding the Base64 string into a PNG and opening it revealed a password written
out as text inside the image itself. Login succeeded with the recovered
credentials.

## File Upload Filter Bypass

The portal offered a file upload feature. Uploading a PHP reverse shell directly
got rejected — the app was filtering by file extension, only allowing
`png`/`jpg`/`gif`.

The fix: rename the shell to spoof a double extension, e.g. `shell.php.jpg`. The
app accepted it as a valid image, and the upload succeeded.

Set up a listener:

```
nc -nvlp 7777
```

Navigating directly to the uploaded file triggered execution and returned a shell
running as the `apache` user. Upgraded it to a proper TTY with the usual
`pty.spawn` trick.

## Enumeration as apache

`/home` contained three users, but only one home directory was readable. Inside it
sat a notes file describing a cron job setup: a script running every minute as a
more privileged user, executing whatever commands were written into a specific
file in `/tmp`.

That's a direct cron abuse opportunity — writing a `chmod` command into the
watched file, then waiting for the cron to fire, opened up access to the more
privileged user's home directory.

## Decoding Credentials from the Admin Directory

That directory held a couple of encoded credential files and, critically, the
Python script used to generate them. Reading the script showed the encoding logic:
Base64-encode the string, reverse it, then apply ROT13.

Reversing that (ROT13 first, then un-reverse, then Base64-decode) recovered two
working plaintext passwords from the encoded files.

## Lateral Movement

Switched to the next user with one of the decoded passwords via `su`. That user's
directory contained a `.bash_history` showing repeated use of a custom binary
being run through `sudo` as yet another user — a clear pointer toward the final
privesc step.

## Privilege Escalation

```
sudo -l
```

Confirmed the current user could run that custom binary as another user via sudo,
with no password. Running it with `/bin/bash` as an argument dropped straight into
a root shell.

## Capturing the Flag

`/root` held the final flag file, confirming full compromise.

## Attack Chain Summary

```
netdiscover → nmap (port 80 only, robots.txt rabbit holes)
  → guessed /fristi admin portal
  → page source: username + Base64-encoded image → password
  → login → file upload → double-extension bypass → shell (apache)
  → notes.txt → cron job abuse → access to privileged user's home
  → custom encoder script reversed → two more passwords
  → su (lateral move) → .bash_history → sudo custom binary
  → sudo -u <user> <binary> /bin/bash → ROOT
```

## Key Takeaways

1. `robots.txt` disallowed entries aren't always the actual target, but they hint
   at naming conventions. The themed pattern here pointed straight at the real
   hidden admin path — use `robots.txt` to understand site structure, not just as
   a checklist of URLs to visit.
2. Always read the full page source. A username and an entire password hidden
   inside an image were sitting in plain HTML — automated tools won't always catch
   this; manual source review is a habit worth keeping.
3. File upload filters based purely on extension are weak. Double extensions,
   case variations, and MIME type mismatches are all worth testing the moment you
   hit an upload restriction.
4. Cron job abuse is a powerful lateral movement technique. Any time a privileged
   cron reads commands from a world-writable file, whoever can write to that file
   can execute as the cron owner — always check scheduled tasks during
   enumeration.
5. Custom encoding scripts are decodable the moment you find them alongside their
   own encoded output — read the logic, invert the operations, decode.
6. `.bash_history` reveals intended workflows. The path to the final privesc
   binary and how to invoke it were both sitting in plain history — never skip it.

## Tools Used

`netdiscover`, `nmap`, `base64`, a PHP reverse shell, `netcat`, Python (for
reversing the custom encoder).
