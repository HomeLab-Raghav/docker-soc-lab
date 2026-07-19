# SSH (Secure Shell)

## What it is

SSH (Secure Shell) is a cryptographic network protocol used to securely operate network
services over an unsecured network — most commonly for **remote command-line login**
and file transfer (via `scp`/`sftp`, which run over the SSH transport). It replaced
legacy, cleartext protocols like Telnet and rlogin, which sent credentials and session
data unencrypted. On Linux, the near-universal implementation is **OpenSSH**, split into
a server daemon (`sshd`) and a client (`ssh`).

In this lab, SSH is the **entire attack surface**: `target-ubuntu` runs `sshd`, exposes
port 22 on the internal Docker network, and the account `victim` was brute-forced over
exactly this protocol.

## How authentication works

SSH supports several authentication methods, configurable (and combinable) per-server
via `sshd_config`:

| Method | How it works | Relative security |
|---|---|---|
| **Password authentication** | Client sends a username + password over the encrypted SSH channel; server checks it against the local user database (PAM) | Weak — vulnerable to brute force / credential guessing (this lab), unless paired with rate limiting or lockout |
| **Public key authentication** | Client proves possession of a private key matching a public key already trusted by the server (`~/.ssh/authorized_keys`), via a cryptographic challenge — the private key itself is never transmitted | Strong — nothing guessable is ever sent over the wire |
| **Keyboard-interactive / 2FA** | Server-driven prompt sequence, often used to layer OTP/TOTP or hardware token challenges on top of password auth | Strong when combined with a second factor |
| **Host-based authentication** | Trust based on the *client machine's* identity (its host key) rather than the user — rarely used, considered legacy | Weak in most modern threat models |
| **GSSAPI / Kerberos** | Delegates authentication to an existing Kerberos realm (common in enterprise/AD environments) | Strong, centrally managed |

This lab's target was deliberately configured with **password authentication enabled**
(`victim:password123`) specifically so a brute-force attack against it would be possible
and detectable — see [`agent.yml`](../../agent.yml) in the repo root for the exact
provisioning script.

### The SSH connection handshake, briefly

1. **TCP connection** established (default port 22).
2. **Protocol/version exchange** — client and server exchange SSH version banners.
3. **Key exchange (KEX)** — client and server negotiate a shared symmetric session key
   using asymmetric cryptography (e.g. Diffie-Hellman), establishing an encrypted
   channel *before* any authentication happens. Everything after this point — including
   password authentication attempts — travels encrypted.
4. **Server host key verification** — the client checks the server's host key against
   its known_hosts file (the "authenticity of host X can't be established" prompt on
   first connection).
5. **Authentication** — one or more of the methods above, attempted in the order
   configured by `AuthenticationMethods`/`PreferredAuthentications`.
6. **Session** — once authenticated, a channel is opened for a shell, command execution,
   port forwarding, or file transfer (SFTP/SCP).

## Ports

| Port | Protocol | Notes |
|---|---|---|
| **22/tcp** | Default SSH port | Used by `target-ubuntu` in this lab |
| *(custom)* | Often changed in production (`Port 2222`, etc.) | "Security by obscurity" — reduces automated internet-wide scanning noise but is not a real control against a targeted attacker |

## Log locations

This is the detail that makes SSH observable to a SIEM, and directly explains why this
lab needed `rsyslog` at all (see [`rsyslog.md`](rsyslog.md)):

| System type | Log location | Notes |
|---|---|---|
| Debian/Ubuntu (traditional syslog) | `/var/log/auth.log` | What this lab's `target-ubuntu` container uses — `sshd` and PAM both write here via `rsyslogd` |
| RHEL/CentOS/Fedora | `/var/log/secure` | Same content, different filename convention |
| systemd-based systems (most modern distros) | `journalctl -u ssh` / `journalctl -u sshd` | Structured, queryable systemd journal instead of (or in addition to) a flat file |
| PAM-specific messages | Same files as above — PAM logs alongside `sshd` since it's the authentication backend `sshd` delegates to | Explains why this lab's alerts include *both* `sshd:` and `PAM:` prefixed log lines for the same login attempt |

A typical raw failed-login line looks like:

```
Jul 19 13:37:43 target-ubuntu sshd[1234]: Failed password for victim from 172.18.0.3 port 51522 ssh2
```

And a successful one:

```
Jul 19 13:37:45 target-ubuntu sshd[1240]: Accepted password for victim from 172.18.0.3 port 51530 ssh2
```

These exact log patterns are what Wazuh's `sshd_rules.xml` decoder/rule set matches
against to raise alerts `5760` (failed) and `5715` (success) — see [`wazuh.md`](wazuh.md).

## Brute-force exposure

SSH is one of the most commonly brute-forced services on the internet, for structural
reasons:

- It's almost always present on any Linux server, and frequently internet-facing.
- Its authentication is fast per-attempt (unlike, say, a rate-limited web login form),
  making high-volume automated guessing cheap for an attacker.
- Password authentication, when enabled, has no inherent limit on attempts unless
  explicitly configured (`MaxAuthTries`) or enforced externally (fail2ban, a WAF, a
  cloud security group).
- Valid usernames are often guessable or public (`root`, `admin`, `ubuntu`, a company's
  naming convention, or — as in this lab — simply known in advance).

This lab demonstrated the textbook version of this exposure: a single known username
(`victim`), password authentication enabled, no rate limiting configured, and an
8-entry wordlist recovered the real password in under 4 seconds. Full attack detail:
[`hydra-lab/incident-response/hydra-lab-1-report.md`](../hydra-lab/incident-response/hydra-lab-1-report.md).

## Hardening

In order of impact, highest first:

1. **Disable password authentication entirely.**
   ```
   # /etc/ssh/sshd_config
   PasswordAuthentication no
   KbdInteractiveAuthentication no
   ```
   This single change makes the entire brute-force attack class in this lab
   structurally impossible — there is no password to guess. Requires deploying public
   keys to every legitimate user first.

2. **Disable root login over SSH.**
   ```
   PermitRootLogin no
   ```
   Removes the single most commonly guessed username from the attack surface entirely.

3. **Enforce key-based auth + a second factor** for anything sensitive (e.g. `sshd`
   `AuthenticationMethods publickey,keyboard-interactive` with a PAM OTP module).

4. **Rate-limit / lock out repeated failures.**
   - `fail2ban` watching `/var/log/auth.log` (or the journal) and firewalling offending
     IPs after N failures in a window — the classic open-source mitigation.
   - Cloud-native equivalents: AWS Security Group + GuardDuty response, Azure NSG +
     Sentinel playbook, etc.
   - PAM's `pam_faillock`/`pam_tally2` modules can lock accounts after N local
     authentication failures.

5. **Restrict source IPs** via firewall rules, a bastion/jump host, or a VPN — SSH
   should rarely be reachable from the entire internet in production.

6. **Change the listening port** (`Port 2222`) — low-value on its own, but reduces
   noise from blind internet-wide scanners, making genuinely targeted attempts easier
   to spot in logs.

7. **Enable verbose logging and ship it to a SIEM** — exactly what this lab
   demonstrates end to end: without centralized, alerting log collection, none of the
   above controls tell you *when* they were tested against.

8. **Set `MaxAuthTries` and `LoginGraceTime` conservatively** in `sshd_config` to limit
   how many auth attempts a single connection can make and how long a connection can
   stay open while unauthenticated.
