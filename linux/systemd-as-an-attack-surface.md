# systemd as an Attack Surface

> **Category:** Linux · Privilege Escalation · System Internals  
> **Level:** Practitioner  

---

Most privilege escalation checklists cover the obvious paths: SUID binaries, sudo misconfigurations, writable `/etc/passwd`, cron jobs running as root. These are well-documented and most hardened systems have them under control.

What gets less attention is the service execution model introduced by systemd — specifically, the combination of services running as root that source or execute files in user-influenced paths.

## The Setup

systemd manages services through unit files. A typical application service might look like:

```ini
[Service]
User=root
ExecStart=/opt/app/scripts/start.sh
```

The script at `/opt/app/scripts/start.sh` runs as root. If that script sources a configuration file, executes a helper, or iterates over a directory — and any part of that path is writable by a non-root user — then privilege escalation is possible without touching a CVE.

## The Pattern

In a real exercise, the chain looked like this:

1. A systemd service ran as root every 5 minutes via a timer
2. The service called a script that sourced files from `/opt/app/config/checks.d/`
3. The `checks.d/` directory was group-writable for `appops`
4. The compromised user was not in `appops` — but had write access to a subdirectory due to inherited permissions from an earlier deployment
5. A file dropped in that subdirectory was sourced by the root script on the next timer tick

The escalation required no exploit. Just a file write and waiting 5 minutes.

## Finding It

The enumeration that surfaces this:

```bash
# List all timers and their associated services
systemctl list-timers --all

# Inspect a service unit
systemctl cat <service-name>

# Trace the execution path
cat /path/to/called/script

# Check every file and directory in the execution path
ls -la /opt/app/scripts/
ls -la /opt/app/config/
ls -la /opt/app/config/checks.d/

# Find what your user can write
find /opt/app -writable 2>/dev/null
```

The key step most people skip: checking **subdirectory permissions recursively**, not just the top-level path named in the unit file.

## The Execution Environment Matters

One important detail: systemd services do not run in the same environment as an interactive shell. `PATH`, `HOME`, and other variables may be different or absent. The script may behave differently than when run manually. This matters when crafting a payload — a reverse shell might fail because `bash` is not in the expected path. Writing to a file or setting a SUID bit is more reliable.

```bash
# Reliable payload for non-interactive root execution
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash

# After the timer fires:
/tmp/rootbash -p
```

## Hardening

systemd provides several directives specifically designed to limit what a service can do:

```ini
[Service]
User=appuser                    # Don't run as root if not needed
NoNewPrivileges=true            # Prevent privilege escalation
ProtectSystem=strict            # Make most of the filesystem read-only
ProtectHome=true                # No access to /home, /root
ReadOnlyPaths=/opt/app/scripts  # Lock the script itself
ReadWritePaths=/opt/app/logs    # Allow only specific write paths
PrivateTmp=true                 # Isolated /tmp
```

These directives do not require application changes — they are applied at the unit file level and constrain what the service can do regardless of what the script does.

## The Broader Point

The vulnerability was not in a package, a kernel version, or a CVE database entry. It was in the permission model of a directory tree that nobody audited after a deployment script ran and set permissions too broadly.

Privilege escalation on modern Linux systems increasingly looks like this: no exploit, no kernel bug — just a service running as root with a writable file somewhere in its execution path, waiting for a timer to fire.

---

*Related: [Case 03 — systemd Service Abuse and Privilege Escalation](https://github.com/juliomauro/sec-writeups/tree/main/cases/03-linux-privesc-systemd)*
