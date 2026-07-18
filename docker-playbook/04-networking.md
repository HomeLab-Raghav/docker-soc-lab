# 04 — Docker Networking

## How containers talk to each other

By default, containers are isolated. They can only communicate if they're on the same Docker network. Docker Compose automatically creates a network for each project and attaches all of that project's containers to it.

In this lab, the main stack's network is called **`docker-soc-lab_default`**. All four containers (manager, indexer, dashboard, and target-ubuntu) are on this network.

---

## The `docker-soc-lab_default` network

When you ran `docker-compose up -d` for the first time, Compose created this network:

```powershell
docker network ls
```

```
NETWORK ID     NAME                      DRIVER    SCOPE
0816988301ba   docker-soc-lab_default    bridge    local
```

It's a **bridge network** — the default type for containers on a single host. Each container gets:
- A private IP address on the `172.x.x.x` range
- DNS resolution by container hostname

That last point is why you see `wazuh.manager` in config files instead of an IP address:

```yaml
# In docker-compose.yml — manager tells Filebeat where the indexer is
- INDEXER_URL=https://wazuh.indexer:9200

# In agent.yml — agent uses the manager's hostname to enroll
WAZUH_MANAGER='wazuh.manager' dpkg -i ./wazuh-agent.deb
```

Docker's built-in DNS resolves `wazuh.manager` to whatever IP the manager container was assigned. You never hardcode IPs.

---

## Two separate compose projects, one shared network

The Wazuh stack (`docker-compose.yml`) and the agent target (`agent.yml`) are two separate compose projects. By default, each project gets its own network. The agent needs to reach the manager — so instead of creating a new network, `agent.yml` joins the existing one:

```yaml
# agent.yml
networks:
  docker-soc-lab_default:
    external: true    # this network already exists — don't create it
```

The `external: true` tells Compose "this network was created by someone else, just use it." If the main stack isn't running (and the network doesn't exist yet), `docker-compose -f agent.yml up -d` will fail with a network not found error.

---

## Port mapping — how traffic gets in from your browser

Containers are isolated from your Windows host by default. To reach a container from your browser, you **publish** a port: `host_port:container_port`.

```yaml
# In docker-compose.yml — wazuh.dashboard
ports:
  - 443:5601
```

This maps:
- **5601** inside the container (where the dashboard process listens)
- **443** on your Windows host (what your browser connects to)

So `https://localhost` → your Windows machine's port 443 → Docker routes it → dashboard container port 5601.

The manager exposes multiple ports for different purposes:

```yaml
# wazuh.manager
ports:
  - "1514:1514"    # Wazuh agent protocol (OSSEC message channel)
  - "1515:1515"    # Agent auto-enrollment (authd)
  - "514:514/udp"  # Syslog ingestion
  - "55000:55000"  # Wazuh REST API
```

The indexer:
```yaml
# wazuh.indexer
ports:
  - "9200:9200"    # OpenSearch HTTP API (used by manager's Filebeat + dashboard queries)
```

---

## DNS names vs IPs in this lab

| Container | Hostname / DNS name | What uses it |
|---|---|---|
| `docker-soc-lab-wazuh.manager-1` | `wazuh.manager` | Agent enrollment, dashboard API calls |
| `docker-soc-lab-wazuh.indexer-1` | `wazuh.indexer` | Manager Filebeat output, dashboard queries |
| `docker-soc-lab-wazuh.dashboard-1` | `wazuh.dashboard` | Internal only |
| `target-ubuntu` | `target-ubuntu` | Referenced in agent.yml `container_name` |

To find the actual IP of any container:

```powershell
docker inspect target-ubuntu --format "{{.NetworkSettings.Networks.docker-soc-lab_default.IPAddress}}"
```

You need the real IP (not the hostname) when you're connecting to the container **from outside Docker** — for example, pointing `hydra` at target-ubuntu's SSH port from a WSL terminal.

---

## Traffic flow for the attack scenario

```
hydra (WSL terminal)
  │
  └── TCP to target-ubuntu IP:22 (SSH brute force)
        │
        └── target-ubuntu sshd logs failed auth → Wazuh agent reads /var/log/auth.log
              │
              └── agent → wazuh.manager:1514 (event forwarded)
                    │
                    └── manager analyses event → fires rule 5712
                          │
                          └── manager → Filebeat → wazuh.indexer:9200 (alert stored)
                                │
                                └── wazuh.dashboard:5601 → your browser (alert visible)
```

All of that flows over the `docker-soc-lab_default` bridge network except the final browser connection, which comes in on port 443.
