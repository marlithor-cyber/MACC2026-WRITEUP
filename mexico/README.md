# Mexico Lab Writeup

## MACC 2026 National CTF Morocco

Presented by Team CyberDune Club and EST Guelmim, Ibn Zohr University

## Executive Summary

The Mexico lab focused on a management-plane compromise of an exposed `Nginx UI` instance. The attack path was short but high impact: an unauthenticated backup disclosure endpoint exposed an encrypted backup together with the material required to decrypt it. The decrypted backup leaked sensitive application secrets, which were then reused to enter the management context and access the built-in terminal as `root`. From there, the final flag was recovered directly from the filesystem.

## Scope

- Ops box: `13.61.142.69`
- Target host: `176.16.61.172`
- Exposed service: `Nginx UI`
- Service port: `18082`
- Objective: recover the flag from `/home/local.txt`

The relevant web interface was reachable at:

```text
http://176.16.61.172:18082
```

## Vulnerability Overview

The exposed management service suffered from an unauthenticated backup disclosure vulnerability tracked in the local notes as:

```text
CVE-2026-27944
```

The vulnerable endpoint was:

```text
/api/backup
```

Two problems existed at the same time:

- The endpoint was accessible without authentication.
- The response included decryption material in the `X-Backup-Security` header.

That combination collapsed the backup protection model completely. Instead of only exposing an encrypted archive, the service disclosed both the backup and the secret needed to decrypt it.

## Attack Path

The exploitation chain was concise and reliable:

1. Access the exposed `Nginx UI` service on port `18082`.
2. Request the unauthenticated `/api/backup` endpoint.
3. Capture the returned backup archive.
4. Extract the `X-Backup-Security` response header.
5. Use the leaked security value to decrypt the backup.
6. Recover the node secret and related session material from the decrypted data.
7. Reuse the leaked secret to access the `Nginx UI` management context.
8. Use the built-in terminal or node access provided by the interface.
9. Read the flag directly as `root`.

This is a good example of a control-plane breach where the application’s own administrative features become the post-exploitation environment.

## Backup Disclosure and Secret Recovery

The decrypted backup exposed sensitive internal `Nginx UI` data, including the node secret used by the platform for trusted management operations. Once that secret was recovered, it became possible to re-enter the application with elevated trust rather than chaining a second memory-corruption or command-injection bug.

This was the critical pivot:

- Initial issue: unauthenticated backup exposure
- Follow-on weakness: secret material stored inside the decrypted backup
- Privilege outcome: administrative access to the management layer

Because the platform already included terminal functionality, secret reuse immediately translated into shell access on the target host.

## Shell Access and Flag Retrieval

The built-in terminal session came up as `root`, which removed the need for a separate local privilege escalation stage.

The final validation step was straightforward:

```bash
cat /home/local.txt
```

Recovered flag:

```text
flag_6446b7e5_8d26_416b_b6b1_c19adbe72749
```

## Root Cause

The compromise was made possible by several security failures in sequence:

- Exposure of an administrative interface to the target network
- Backup download functionality accessible without authentication
- Disclosure of backup decryption material in an HTTP response header
- Presence of reusable node or session secrets inside the decrypted backup
- Availability of a privileged built-in terminal in the management panel

None of these controls should have aligned this cleanly in a real deployment. Together, they created a direct path from anonymous web access to `root` shell access.

## Defensive Recommendations

- Never expose administrative interfaces such as `Nginx UI` directly without strict access controls.
- Require authentication and authorization for backup functionality.
- Do not transmit backup decryption secrets in response headers or alongside encrypted artifacts.
- Separate operational secrets from user-exportable backup data where possible.
- Restrict or disable in-panel terminal access unless it is strictly necessary and strongly audited.
- Monitor for unusual requests to administrative backup endpoints and secret-bearing API responses.

## Conclusion

The Mexico lab highlighted how dangerous management-plane bugs can be when backup and administrative features are not isolated properly. A single unauthenticated endpoint provided both the protected data and the means to decrypt it, which in turn exposed trusted secrets and unlocked a root shell through the platform’s own terminal capability. The path from disclosure to full system compromise was short, deterministic, and operationally realistic.
