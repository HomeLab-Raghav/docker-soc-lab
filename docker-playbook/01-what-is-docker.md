# 01 — What is Docker?

## The short version

Docker lets you package an application and everything it needs (OS libraries, config, dependencies) into a single unit called a **container**. You can spin one up in seconds, throw it away when you're done, and it'll behave identically on any machine that has Docker installed.

---

## Containers vs Virtual Machines

The traditional way to run an isolated environment is a virtual machine (VM): you emulate an entire computer, including a full operating system kernel. That's powerful but slow — a VM on 16 GB RAM might be able to run 2–3 before it grinds.

Containers share the host OS kernel. They're just isolated processes — isolated from each other, but not virtualising a whole machine. That makes them:

| | VMs | Containers |
|---|---|---|
| Start time | Minutes | Seconds |
| Size on disk | GBs (full OS) | MBs (only what's needed) |
| Isolation | Full kernel separation | Process + filesystem isolation |
| Overhead | High (hypervisor) | Near-zero |

**In this lab:** the Wazuh manager, indexer, and dashboard are three Docker containers. Standing them all up takes under 2 minutes including image downloads. Doing the same with VMs would take most of an afternoon and need far more RAM.

---

## Images vs Containers

These two terms trip people up at first.

**Image** — a read-only blueprint. It defines what software is installed, what files exist, what the entry point command is. Think of it like a `.iso` file or a VM snapshot.

**Container** — a running instance of an image. You can run many containers from the same image simultaneously. Each gets its own writable filesystem layer on top of the read-only image.

```
wazuh/wazuh-manager:4.9.0  ←  this is the IMAGE  (blueprint)
         │
         ├── docker-soc-lab-wazuh.manager-1  ←  this is the CONTAINER  (running instance)
```

In this lab we use four images:
- `wazuh/wazuh-manager:4.9.0`
- `wazuh/wazuh-indexer:4.9.0`
- `wazuh/wazuh-dashboard:4.9.0`
- `ubuntu:22.04` (for target-ubuntu)

Each becomes exactly one container.

---

## Why Docker for this lab?

Setting up a Wazuh SIEM stack manually — installing OpenSearch, Wazuh manager, Wazuh dashboard, wiring them together with TLS — takes days of reading documentation and fighting dependencies. Wazuh's official Docker setup does all of that for you in a single file.

More practically: I can `docker-compose down` to tear the whole lab down, `docker-compose up -d` to bring it back, and the state is preserved in named volumes. That's not something you can do cleanly with a hand-installed stack.

The `target-ubuntu` container takes this further — it boots a fresh Ubuntu, installs SSH and the Wazuh agent, and connects to the manager, all in one `docker-compose up -d`. No ISOs, no installer wizards.

---

## The mental model for this lab

```
Your Windows machine
│
└── Docker Desktop (runs a hidden Linux VM via WSL2)
    │
    └── docker-soc-lab_default network
        ├── wazuh.manager   (container)
        ├── wazuh.indexer   (container)
        ├── wazuh.dashboard (container, port 443 exposed to Windows)
        └── target-ubuntu   (container, runs sshd)
```

Your browser hits `https://localhost:443` → Docker routes it to the dashboard container's port 5601.

---

## Next

[02-essential-commands.md](02-essential-commands.md) — the commands you'll use every day.
