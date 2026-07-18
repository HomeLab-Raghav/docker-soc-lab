# Docker Playbook

A hands-on Docker reference written around the commands and concepts I actually used building the Wazuh SIEM lab. Each file focuses on one area, with real examples from this project.

---

## Table of contents

| File | What it covers |
|---|---|
| [01-what-is-docker.md](01-what-is-docker.md) | Containers vs VMs, images vs containers, why Docker for this lab |
| [02-essential-commands.md](02-essential-commands.md) | `docker ps`, `exec`, `logs`, `inspect`, `restart`, `rm` — syntax + real lab examples |
| [03-docker-compose.md](03-docker-compose.md) | What compose is, our compose files explained line by line, `up`/`down`/`ps` |
| [04-networking.md](04-networking.md) | How the containers talk to each other, the `docker-soc-lab_default` network, port mapping |
| [05-volumes-and-persistence.md](05-volumes-and-persistence.md) | Named volumes, why data survives restarts, what happens without them |
| [06-troubleshooting.md](06-troubleshooting.md) | Real issues I hit during the build and how I fixed them |

---

Start with [01-what-is-docker.md](01-what-is-docker.md) if you're new to containers, or jump straight to [06-troubleshooting.md](06-troubleshooting.md) if something broke.
