# Hydra (THC-Hydra)

## What it is

Hydra — often called **THC-Hydra** after its origin at "The Hacker's Choice" — is an
open-source, highly parallelized **online password-guessing / credential brute-forcing
tool**. It's one of the oldest and most widely used tools in offensive security tooling,
maintained on GitHub at [`vanhauser-thc/thc-hydra`](https://github.com/vanhauser-thc/thc-hydra),
and ships pre-installed on Kali Linux and most penetration-testing distributions.

Unlike an *offline* password cracker (like Hashcat or John the Ripper, which crack
password **hashes** you already possess), Hydra performs **online** attacks: it actively
connects to a live service over the network and submits real login attempts, one
credential pair at a time, watching the service's response to determine success or
failure. This is exactly what happened in this lab — Hydra opened real SSH connections
to `target-ubuntu` and tried real logins.

## How it works

1. **You give it a target** — a hostname/IP, or a file of many targets (`-M`).
2. **You give it credentials to try** — a single username or list (`-l`/`-L`), and a
   single password or list (`-p`/`-P`), or a combination file (`-C user:pass`).
3. **You give it a protocol module** — Hydra supports 50+ protocols via pluggable
   modules: `ssh`, `ftp`, `telnet`, `smb`, `rdp`, `mysql`, `postgres`, `http-get`,
   `http-post-form`, `smtp`, `vnc`, `snmp`, and many more.
4. **Hydra opens multiple parallel connections** (`-t`, default 16, capped lower for
   protocols like SSH that are expensive per-connection) and, for each connection, sends
   one credential pair and inspects the protocol-specific response to classify it as a
   success or failure.
5. **On success**, it prints the valid credential pair and (by default) continues
   trying remaining combinations unless told to stop early (`-f`/`-F`).

## Command used in this lab

```bash
hydra -l victim -P pwlist.txt target-ubuntu ssh -t 4
```

| Part | Meaning |
|---|---|
| `-l victim` | Attack a single, known username: `victim` |
| `-P pwlist.txt` | Try every password in this file, one per line |
| `target-ubuntu` | Target host (resolved via Docker's internal DNS, since the attacker container shared the target's network) |
| `ssh` | Protocol module to use |
| `-t 4` | Run 4 parallel connection attempts at once |

The wordlist (`pwlist.txt`) was built with:

```bash
printf 'admin\n123456\npassword\nletmein\nqwerty\nroot\ntoor\npassword123\n' > pwlist.txt
```

Eight common weak/default passwords, with the real password (`password123`) included so
the attack had a guaranteed, detectable success.

## Full flag reference

### Credential input

| Flag | Purpose |
|---|---|
| `-l LOGIN` | Single username to try |
| `-L FILE` | File of usernames to try (one per line) |
| `-p PASS` | Single password to try |
| `-P FILE` | File of passwords to try (one per line) |
| `-C FILE` | Combined `login:pass` file — skips the cartesian product of `-L`/`-P` in favor of exact pairs |
| `-e nsr` | Additional password checks: `n` = null password, `s` = login as password, `r` = reversed login as password |
| `-x MIN:MAX:CHARSET` | Generate passwords algorithmically instead of from a file (brute-force character generation, e.g. `-x 4:6:a` for 4–6 lowercase letters) |
| `-u` | Loop users first, then passwords (default is the reverse) — changes attack pacing per-account, useful for evading simple per-user lockout thresholds |

### Target specification

| Flag | Purpose |
|---|---|
| `server` (positional) | Single target hostname/IP |
| `-M FILE` | File of multiple targets to attack (e.g. sweeping a whole subnet) |
| `-s PORT` | Non-default port for the service |
| `-S` | Use SSL/TLS for the connection where the module supports it |

### Performance & behavior

| Flag | Purpose |
|---|---|
| `-t N` | Number of parallel connections/tasks (default 16; lower for slow/rate-limited protocols like SSH) |
| `-T N` | Number of parallel **hosts** to attack at once, when using `-M` |
| `-w SEC` | Wait time between connection attempts (throttling, useful to dodge lockout/rate-limiting) |
| `-W SEC` | Wait time for a response before timing out |
| `-c SEC` | Wait time between login attempt reconnects |
| `-f` | Stop attacking a **single host** as soon as one valid credential pair is found |
| `-F` | Stop attacking **all hosts** as soon as one valid credential pair is found anywhere |
| `-I` | Ignore an existing restore file and start a fresh session |

### Output

| Flag | Purpose |
|---|---|
| `-o FILE` | Write found credentials to a file |
| `-b FORMAT` | Output format for `-o` (`text`, `json`, `jsonv1`) |
| `-v` / `-V` | Verbose mode — show each login attempt as it happens |
| `-d` | Debug output |
| `-q` | Quiet mode — suppress connection error messages |

### Modules and usage

```bash
hydra -U             # list all supported protocol modules
hydra -U ssh          # show module-specific options for ssh
```

## Real example commands (reference)

```bash
# Single user, password list, 4 parallel tasks (this lab)
hydra -l victim -P pwlist.txt target-ubuntu ssh -t 4

# Stop as soon as a valid credential is found
hydra -l victim -P pwlist.txt -f target-ubuntu ssh

# Full user list x password list (cartesian product) against FTP
hydra -L users.txt -P rockyou.txt ftp.example.com ftp

# HTTP POST login form brute force
hydra -l admin -P passwords.txt www.example.com http-post-form \
  "/login:username=^USER^&password=^PASS^:Invalid credentials"

# RDP brute force with throttling to avoid lockout
hydra -l administrator -P passwords.txt -t 1 -w 5 rdp://10.0.0.5

# Resume a previously interrupted session
hydra -R
```

## Use cases

- **Authorized penetration testing / red team engagements** — validating whether weak
  or default credentials exist on in-scope services.
- **Security lab training** (this repo) — generating realistic brute-force traffic to
  validate SIEM/IDS detection coverage.
- **Password policy validation** — confirming an organization's password complexity
  rules actually resist common wordlists (e.g. rockyou.txt, SecLists).
- **CTF competitions** — a staple tool for "guess the login" style challenges.

Hydra is a **dual-use** tool: the exact same capability used here for authorized,
defensive detection testing is what an attacker uses in a real credential-stuffing or
password-spraying attack. Always confirm explicit authorization before running Hydra
against any system you don't own or aren't contracted to test.

## How to detect Hydra (and brute-force tools generally) defensively

| Signal | Why it works |
|---|---|
| **Burst of auth failures from one source IP in a short window** | The core signature — this lab's entire 8-password attack completed in ~6.5 seconds; the built-in Wazuh correlation rules (`5551`, `5712`, `5720`) are designed to catch exactly this pattern (see [`wazuh.md`](wazuh.md)) |
| **Many failures for the same username, few/no failures for other usernames** | Distinguishes targeted brute force from a password-spray (many usernames, few passwords each) |
| **Uniform timing between attempts** | Human logins have irregular timing; scripted tools often show suspiciously even spacing or, as in this lab, tightly clustered parallel bursts |
| **Connections from unusual source ASNs / geographies / known-Tor or VPS ranges** | Hydra is frequently run from cloud/VPS infrastructure rather than residential IPs |
| **SSH client banner / version fingerprint** | Some brute-force tooling uses distinctive SSH client version strings, though Hydra's SSH module typically negotiates like a normal OpenSSH client |
| **A success immediately following a run of failures for the same account** | The strongest single indicator of a *successful* brute force, as reconstructed in this lab's [incident report](../hydra-lab/incident-response/hydra-lab-1-report.md) — Wazuh rule `5715` right after a cluster of `5760`/`5503` failures |
| **Fail2ban / SSH rate limiting** | Preventive control: after N failures from one IP within a window, block it at the firewall — turns detection into automatic containment |

See [`ssh.md`](ssh.md) for SSH-specific hardening that neutralizes this entire attack
class (key-based auth, disabling password login) regardless of how fast or slow the
brute-force tool is.
