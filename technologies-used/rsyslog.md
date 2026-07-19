# rsyslog

## What it is

`rsyslog` ("rocket-fast syslog") is a high-performance implementation of the **syslog
protocol** for Unix-like systems — a daemon that receives log messages from the kernel
and applications (via the standard `/dev/log` socket or the syslog network protocol),
and routes them to one or more destinations: local files, remote log servers, databases,
or other processing pipelines. It's the default system logger on Debian and Ubuntu (the
base OS used by `target-ubuntu` in this lab) and one of the most widely deployed syslog
daemons in production Linux environments generally.

## Why it mattered in this lab

`target-ubuntu` is a **minimal `ubuntu:22.04` container** — it does not run `systemd`
and therefore has no `journald` (systemd's modern structured logging service) to receive
and store log messages. Without *something* listening on `/dev/log`, any process calling
the standard `syslog()` C library function — which is exactly what `sshd` and PAM do
internally to log authentication events — has nowhere for those messages to go; they are
silently dropped.

This is precisely why [`agent.yml`](../../agent.yml)'s bootstrap script explicitly
installs and starts `rsyslog` *before* configuring the Wazuh agent:

```bash
apt-get install -y curl gnupg2 openssh-server rsyslog sudo lsb-release &&
mkdir -p /run/sshd &&
rsyslogd &&
# ... wazuh agent install ...
```

Once `rsyslogd` is running, it owns `/dev/log` and, per its default configuration,
routes `auth`/`authpriv` facility messages (which is what `sshd` and PAM use) into
`/var/log/auth.log`. **Without this step, none of the SSH brute-force alerts in this
lab would have existed** — not because Wazuh failed, but because the log data itself
would never have been written anywhere for the Wazuh agent to read.

## How logging works on Linux

Linux logging has evolved through (and today often runs) three overlapping layers:

### 1. The syslog protocol/API (classic)

Applications call the C library's `syslog()` function (or write directly to the
`/dev/log` Unix socket) with a message, a **facility** (what kind of thing is logging —
`auth`, `authpriv`, `daemon`, `kern`, `mail`, `cron`, `local0`–`local7`, etc.) and a
**severity** (`emerg`, `alert`, `crit`, `err`, `warning`, `notice`, `info`, `debug`). A
syslog daemon (rsyslog, syslog-ng, or journald in compatibility mode) receives these and
routes them per its configuration.

### 2. `journald` (modern, systemd-based distros)

On most current distros (full Ubuntu/Debian installs, RHEL, Fedora, etc.), `systemd`'s
`journald` is the primary log receiver, storing structured, indexed, binary-format logs
queryable with `journalctl`. `journald` can also forward to `rsyslog` for
backward-compatible flat-file output, and `rsyslog` can itself read from the journal —
the two are commonly run together, not as strict alternatives.

### 3. rsyslog (flat-file / routing layer)

`rsyslog` receives messages (either directly via `/dev/log`, or forwarded from
`journald`) and applies rules to route them — by default, to flat text files under
`/var/log/`, but also optionally to remote syslog servers (a very common way to
centralize logs for a SIEM), databases, or custom output modules.

## auth.log vs syslog vs journald

| Source | Typical path | Contents | Relevance to this lab |
|---|---|---|---|
| **`/var/log/auth.log`** | Debian/Ubuntu, written by rsyslog from the `auth`/`authpriv` facility | SSH logins, `sudo` usage, PAM authentication events, user/group management | **The file this lab's Wazuh agent tails** — every alert in this lab originated here |
| **`/var/log/syslog`** | Debian/Ubuntu, written by rsyslog from most other facilities | General system messages — services starting/stopping, cron, kernel messages not sent elsewhere | Not used for detection in this lab, but the default catch-all on a full Ubuntu system |
| **`journalctl` (journald)** | Binary, structured, systemd-managed | Everything journald receives — often a superset of what ends up in flat files, on systems that run systemd | **Not present** in this lab's minimal container (no systemd) — this is exactly the gap rsyslog fills |
| **`/var/log/secure`** | RHEL/CentOS/Fedora equivalent of `auth.log` | Same purpose, different distro naming convention | Not applicable to this Ubuntu-based lab, but worth knowing when adapting this lab to a Red Hat–family target |

## The Wazuh agent's configuration for this file

To read `/var/log/auth.log`, the agent's `ossec.conf` needs an explicit `<localfile>`
block — added by `agent.yml`'s bootstrap script if not already present:

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>
```

`<log_format>syslog</log_format>` tells the agent to parse each line using Wazuh's
built-in syslog-format pre-decoder (extracting timestamp, hostname, and program name)
before handing the remainder to the more specific `sshd`/`pam` decoders and rules
described in [`wazuh.md`](wazuh.md).

## Why this matters beyond this lab

Any time a monitored host is a **minimal container, embedded system, or stripped-down
image without systemd/journald**, the same gap applies: application-level `syslog()`
calls (SSH, cron, sudo, many daemons) go nowhere unless something is running to own
`/dev/log`. This is a common, easy-to-miss failure mode when building detection labs or
onboarding lightweight/containerized assets into a SIEM — the agent can be perfectly
configured and the manager's rules perfectly correct, and you'll still see zero alerts,
because the log data was never written to disk in the first place. Always verify with
`ls -la /var/log/auth.log` (or equivalent) and a manual `tail -f` while triggering a test
event, before assuming a detection rule itself is broken.
