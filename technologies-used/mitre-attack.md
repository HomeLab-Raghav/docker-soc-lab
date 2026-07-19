# MITRE ATT&CK

## What the framework is

MITRE ATT&CK (**Adversarial Tactics, Techniques, and Common Knowledge**) is a globally
maintained, publicly available knowledge base of real-world adversary behavior,
published by MITRE Corporation. It catalogs *how* attackers actually operate — observed
from real intrusions and threat intelligence — organized into a matrix that security
teams use as a common vocabulary for describing, detecting, and defending against
attacks. It is the de facto industry standard for threat modeling, detection engineering,
and red team/blue team communication, and is what Wazuh (and virtually every modern SIEM)
uses to automatically classify alerts.

Multiple matrices exist for different domains — **Enterprise** (traditional IT), **Mobile**,
and **ICS** (industrial control systems). This lab's alerts map to the Enterprise matrix.

## Tactics vs. techniques

This distinction is the core of how ATT&CK is organized, and it's worth being precise
about:

| Concept | Definition | Analogy |
|---|---|---|
| **Tactic** | The attacker's **goal** — *why* they're doing something | The destination |
| **Technique** | **How** they achieve that goal — a specific method | The route taken to get there |
| **Sub-technique** | A more specific variant of a technique | A specific turn-by-turn path along that route |

The Enterprise matrix defines 14 tactics, ordered roughly (though not strictly linearly)
along a typical intrusion lifecycle:

```
Reconnaissance → Resource Development → Initial Access → Execution → Persistence →
Privilege Escalation → Defense Evasion → Credential Access → Discovery →
Lateral Movement → Collection → Command and Control → Exfiltration → Impact
```

Each tactic contains many techniques (and many techniques contain sub-techniques). A
single observed action — like a successful SSH login using a guessed password — is
often mapped to **more than one** tactic/technique simultaneously, because it serves
multiple adversary goals at once. This is exactly what happened in this lab: Wazuh
tagged the same successful-login alert with both a **Credential Access** technique
(the password was obtained) and a **Lateral Movement** technique (an interactive remote
session was opened) — see [`wazuh.md`](wazuh.md) and the
[incident report's MITRE section](../hydra-lab/incident-response/hydra-lab-1-report.md#7-mitre-attck-mapping).

## T1110 — Brute Force, in detail

**Tactic:** Credential Access
**Technique ID:** T1110

### Definition

Adversaries may use brute force techniques to gain access to accounts when passwords
are unknown, or when password hashes are obtained but cracking is impractical.
Without knowledge of the password for an account or set of accounts, an adversary may
systematically guess the password using a repetitive or iterative mechanism, optionally
informed by external knowledge (e.g. common password lists, organizational naming
conventions, or previously breached credentials).

### Sub-techniques

| ID | Name | Description | Relevance to this lab |
|---|---|---|---|
| **T1110.001** | Password Guessing | Attempting a list of common/likely passwords against one or a small number of known accounts | **Exactly this lab's attack** — a fixed 8-entry wordlist against the single known account `victim` |
| **T1110.002** | Password Cracking | Attacking a **hash** offline (not applicable here — Hydra performed live/online guessing, not offline hash cracking) | Not used in this lab |
| **T1110.003** | Password Spraying | The inverse of password guessing: **one or a few passwords** tried against **many** accounts, to avoid per-account lockout thresholds | Not used in this lab (single account targeted) but a natural "next lab" variant |
| **T1110.004** | Credential Stuffing | Using **previously breached, known-valid** username/password pairs against a *different* service, betting on password reuse | Not used in this lab (wordlist was generic weak passwords, not a breach dump) |

### Why T1110.001 specifically applies here

This lab's Hydra command — `hydra -l victim -P pwlist.txt target-ubuntu ssh -t 4` —
targets exactly **one** known username with a **static list** of candidate passwords,
which is the textbook definition of T1110.001 (Password Guessing) as distinct from its
sibling sub-techniques. Wazuh's `sshd_rules.xml` and `pam_rules.xml` map their
failed/brute-force-correlation rules (`5710`, `5712`, `5716`, `5720`, `5760`, `5503`,
`5551`) to the parent technique `T1110` (and `5760` specifically to the `.001`
sub-technique), which is why every alert in this lab's timeline carries this mapping.

### Detection guidance (per MITRE ATT&CK's own recommendations)

- Monitor authentication logs for **failure patterns** that may indicate brute forcing:
  many failed attempts in a short window, especially against the same account or from
  the same source.
- Monitor for **anomalously many authentication attempts** across multiple accounts from
  a single source in a short time frame (distinguishing spraying from guessing).
- Consider **account lockout policies** balanced against denial-of-service risk (an
  attacker could deliberately trigger lockouts as a DoS against legitimate users).
- This is precisely the detection logic Wazuh's correlation rules (`5551`, `5712`,
  `5720`) implement natively — frequency + timeframe + same-source-IP conditions —
  described in [`wazuh.md`](wazuh.md).

### Mitigation guidance

- **Multi-factor authentication** — even a fully guessed password is insufficient
  without a second factor.
- **Account lockout / rate limiting** after N failed attempts.
- **Strong password policies** — length and complexity requirements that make wordlist
  attacks impractical.
- **Disable unnecessary authentication methods** — e.g. disabling SSH password auth
  entirely in favor of keys, as covered in [`ssh.md`](ssh.md).
- **Privileged Account Management** — ensure high-value accounts (root, admin,
  service accounts) have the strongest controls, since they're the highest-value brute
  force targets.

## Why this framework matters for this lab

Mapping this lab's Hydra attack to T1110 (and its Lateral Movement / Valid Accounts
neighbors, T1021.004 and T1078) does more than add a label — it means the exact same
detection built here (Wazuh rules on `sshd`/PAM logs) is describable in a vocabulary
that any other analyst, tool, or threat intelligence feed in the industry also uses.
A detection engineer reading "this environment has coverage for T1110.001" immediately
knows what class of attack is covered, without needing to read this lab's specific
Hydra command — which is the entire point of a shared framework.
