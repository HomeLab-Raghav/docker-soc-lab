# 03 — Docker Compose

## What is Docker Compose?

Docker Compose lets you define multiple containers, their configuration, how they connect to each other, and their volumes — all in a single YAML file. Instead of running five `docker run` commands with pages of flags, you run one command and Compose handles everything.

In this lab, `docker-compose up -d` replaces what would otherwise be:
- Pulling three images
- Creating thirteen named volumes
- Creating a Docker network
- Starting three containers with the right environment variables, port mappings, volume mounts, and startup order

One command. One file.

---

## This lab has three compose files

| File | Purpose |
|---|---|
| `docker-compose.yml` | The main Wazuh stack (manager + indexer + dashboard) |
| `generate-indexer-certs.yml` | One-shot cert generator — run once, then never again |
| `agent.yml` | The target-ubuntu attack container — separate project |

The reason for three files: the cert generator and the agent are separate concerns. You don't want `docker-compose up` to run the cert generator every time, and the agent is intentionally a second compose "project" so it can be managed independently.

---

## The main `docker-compose.yml` — annotated

```yaml
version: '3.7'

services:
  wazuh.manager:
    image: wazuh/wazuh-manager:4.9.0   # exact image and version to pull from Docker Hub
    hostname: wazuh.manager             # DNS name on the Docker network
    restart: always                     # restart if it crashes or the host reboots
    ports:
      - "1514:1514"    # agent event port (host:container)
      - "1515:1515"    # agent enrollment port
      - "55000:55000"  # Wazuh REST API
    environment:
      - INDEXER_URL=https://wazuh.indexer:9200
      - INDEXER_USERNAME=admin
      - INDEXER_PASSWORD=SecretPassword
    volumes:
      - wazuh_logs:/var/ossec/logs       # named volume — survives restarts
      - wazuh_etc:/var/ossec/etc         # named volume
      - ./config/wazuh_indexer_ssl_certs/root-ca-manager.pem:/etc/ssl/root-ca.pem
      #   ↑ bind mount — ./config/... on host → /etc/ssl/... in container

  wazuh.indexer:
    image: wazuh/wazuh-indexer:4.9.0
    hostname: wazuh.indexer
    restart: always
    ports:
      - "9200:9200"
    environment:
      - "OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g"   # heap size for OpenSearch

  wazuh.dashboard:
    image: wazuh/wazuh-dashboard:4.9.0
    hostname: wazuh.dashboard
    restart: always
    ports:
      - 443:5601    # expose container's 5601 as host's 443 → https://localhost
    environment:
      - INDEXER_USERNAME=admin
      - INDEXER_PASSWORD=SecretPassword
      - WAZUH_API_URL=https://wazuh.manager    # dashboard talks to manager API
      - DASHBOARD_USERNAME=kibanaserver
      - DASHBOARD_PASSWORD=kibanaserver
    depends_on:
      - wazuh.indexer    # compose won't start dashboard until indexer container is up

volumes:                  # declare all named volumes at the bottom
  wazuh_etc:
  wazuh_logs:
  wazuh_queue:
  wazuh-indexer-data:
  # ... (13 total)
```

### Two types of volume mounts

**Bind mounts** (`./config/file.pem:/path/in/container`) — mount a specific file or folder from your host into the container. Used here for SSL certs and config files that you own and control.

**Named volumes** (`wazuh_logs:/var/ossec/logs`) — Docker manages the storage location. Used for data you want to persist (logs, indexer data, Wazuh queue) without caring exactly where on the host it lives.

---

## `generate-indexer-certs.yml` — annotated

```yaml
version: '3'

services:
  generator:
    image: wazuh/wazuh-certs-generator:0.0.2
    hostname: wazuh-certs-generator
    volumes:
      - ./config/wazuh_indexer_ssl_certs/:/certificates/   # output goes here
      - ./config/certs.yml:/config/certs.yml               # cert definitions go in here
```

No ports, no `restart`. This container does one job (generate certs into `./config/wazuh_indexer_ssl_certs/`) and exits. The `--rm` flag in the run command deletes it after:

```powershell
docker-compose -f generate-indexer-certs.yml run --rm generator
```

---

## `agent.yml` — annotated

```yaml
services:
  target-ubuntu:
    image: ubuntu:22.04
    hostname: target-ubuntu
    container_name: target-ubuntu    # explicit name so docker exec commands are predictable
    networks:
      - docker-soc-lab_default       # join the existing stack network
    command: >
      bash -c "
      apt-get update &&
      apt-get install -y curl gnupg2 openssh-server sudo lsb-release &&
      mkdir -p /run/sshd &&
      curl -so wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.9.0-1_amd64.deb &&
      WAZUH_MANAGER='wazuh.manager' dpkg -i ./wazuh-agent.deb &&
      /var/ossec/bin/wazuh-control restart &&
      (id victim || useradd -m -s /bin/bash victim) &&
      echo 'victim:password123' | chpasswd &&
      /usr/sbin/sshd -D
      "

networks:
  docker-soc-lab_default:
    external: true    # tells compose this network already exists — don't create it
```

The `command:` block is the entire agent bootstrap: install dependencies, download the agent `.deb`, install it with `WAZUH_MANAGER` set so it knows where to enroll, create the `victim` user, then start `sshd` in the foreground (`-D`) so the container stays alive.

The `(id victim || useradd ...)` pattern makes user creation idempotent — it won't fail on `docker restart` because `victim` already exists.

---

## Core compose commands

```powershell
# Start all services in the background
docker-compose up -d

# Start and watch the logs (foreground)
docker-compose up

# Stop all services (containers stop, volumes and network kept)
docker-compose down

# Stop and delete volumes too (wipes all Wazuh data)
docker-compose down -v

# Rebuild/recreate one container without touching the others
docker-compose up -d --force-recreate target-ubuntu

# See status of compose-managed containers
docker-compose ps

# See logs for all services
docker-compose logs -f

# Use a non-default compose file
docker-compose -f agent.yml up -d
docker-compose -f agent.yml down
```

---

## Startup order

`depends_on` controls the order Compose starts containers in. In `docker-compose.yml`:

```yaml
wazuh.dashboard:
  depends_on:
    - wazuh.indexer
```

This means the dashboard container won't start until the indexer container has started. It does **not** wait for the indexer to be healthy/ready — just started. That's why you still need to wait 60–90 seconds after `docker-compose up -d` before the dashboard UI is usable.
