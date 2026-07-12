# Understanding Docker — Why This Lab Uses It

A plain-English explainer of what Docker is and why this SOC lab is built on it.
Written as a learning reference — and as the answer to the interview question
*"walk me through why you used Docker."*

---

## The problem Docker solves

Set up Wazuh perfectly on one laptop and it works. Try to run it on a server, hand it
to a coworker, or rebuild it after wiping your machine, and historically it breaks — a
different OS version, a missing library, a slightly different config. The classic
developer complaint: *"but it works on my machine."*

Docker eliminates that. It packages an application **together with everything it needs
to run** — OS libraries, dependencies, config — into one sealed unit called a
**container**. That container then runs **identically on any machine that has Docker**:
your laptop, a server, a coworker's PC, the cloud.

---

## The shipping-container analogy

Before standardized shipping containers, loading a cargo ship was chaos — every item a
different size, packed by hand, slow and unreliable. Then the world agreed on one
standard steel box. Suddenly it didn't matter *what was inside*; cranes, ships, and
trucks all handled the box the same way.

Docker does this for software. It doesn't matter whether the container holds Wazuh, a
Python app, or a database — Docker runs every container the same standard way. (That's
why the logo is a whale carrying containers.)

---

## Container vs Virtual Machine

| | Virtual Machine | Container |
|---|---|---|
| **What it runs** | An entire guest operating system | Just the app + its direct dependencies |
| **Weight** | Gigabytes | Megabytes |
| **Boot time** | Minutes | Seconds |
| **Isolation** | Full (own OS) | Strong (shares host kernel) |
| **Example** | A VirtualBox AD homelab | This Wazuh SOC lab |

**Houses vs apartments:** VMs are separate houses — each with its own foundation,
plumbing, and electrical system. Independent, but expensive to build. Containers are
apartments in one building — they share the foundation and utilities (the host kernel)
but each has its own locked door and private space. Both give isolation; containers just
don't rebuild the whole building every time.

> **Interview line:** *"I've used both — VMs in VirtualBox for my Active Directory
> homelab, and Docker containers for my SOC lab — so I understand the tradeoff: VMs give
> full isolation with a complete OS each, while containers are lighter and reproducible
> because they share the host kernel."*

---

## Why this SOC lab specifically uses Docker

1. **Reproducibility.** The entire four-service SIEM stack is defined in one text file
   (`docker-compose.yml`). Anyone can clone the repo and run `docker-compose up` to get
   the *exact* same lab. That's the difference between *"here's a screenshot of my lab"*
   and *"here's a lab you can run yourself."*

2. **Speed and disposability.** The indexer, manager, dashboard, and Ubuntu target all
   spin up in minutes. Break something? `docker-compose down` then `up` — fresh in
   seconds. You can experiment fearlessly because nothing is precious.

3. **It's how the industry works now.** Modern security tooling, cloud infrastructure,
   and CI/CD all run on containers. Docker fluency signals you can work in a real modern
   environment, not just click through GUIs.

---

## The three words you'll hear constantly

| Term | What it is | Analogy |
|---|---|---|
| **Image** | A frozen blueprint of an app + its dependencies (e.g. `wazuh/wazuh-manager:4.9.0`) | The recipe |
| **Container** | A running instance of an image | The cooked meal |
| **Docker Compose** | A tool to define and run *multiple* containers together from one file | The head chef running the whole kitchen |

One image can spawn many containers. This lab's `docker-compose.yml` describes all four
services and wires them together, so a single command runs the whole team.

---

## The one-paragraph interview answer

> *"Docker is a containerization platform. Instead of installing software directly on a
> machine and hoping the environment matches, you package the app with all its
> dependencies into a container that runs identically anywhere. It's lighter than a VM
> because containers share the host kernel instead of running a full guest OS. I used it
> for my SOC lab so the entire Wazuh SIEM stack — indexer, manager, dashboard, and a
> monitored target — is defined in a single compose file and reproducible with one
> command, which makes it something anyone can actually run rather than just look at."*

---

## Author

**Raghav Mahajan** — Cybersecurity Analyst (Blue Team / SOC)
GitHub: [github.com/HomeLab-Raghav](https://github.com/HomeLab-Raghav) · Portfolio: [raghv.dev](https://raghv.dev)
