# 02 — Essential Docker Commands

Every command here is one I actually ran while building or debugging this lab.

---

## `docker ps` — list running containers

**What it does:** shows every container that is currently running, with its ID, image, status, and exposed ports.

```powershell
docker ps
```

Add `--format` to cut the noise:

```powershell
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**Lab example output:**
```
NAMES                              STATUS          PORTS
target-ubuntu                      Up 24 minutes
docker-soc-lab-wazuh.dashboard-1   Up 2 hours      0.0.0.0:443->5601/tcp
docker-soc-lab-wazuh.indexer-1     Up 2 hours      0.0.0.0:9200->9200/tcp
docker-soc-lab-wazuh.manager-1     Up 2 hours      0.0.0.0:1514->1514/tcp, ...
```

Add `-a` to see stopped containers too:

```powershell
docker ps -a
```

**When to use it:** first command to run any time something seems wrong. If a container isn't in `docker ps` output, it exited — use `docker ps -a` to see it and check its exit code.

---

## `docker logs` — see a container's output

**What it does:** prints everything the container has written to stdout/stderr.

```powershell
docker logs <container-name>
docker logs --tail 50 <container-name>    # last 50 lines
docker logs -f <container-name>           # follow (live stream)
```

**Lab example — checking why the indexer exited:**
```powershell
docker logs docker-soc-lab-wazuh.indexer-1 --tail 20
```

If you see `max virtual memory areas vm.max_map_count [65530] is too low`, the fix is the WSL `sysctl` command in the README.

**When to use it:** when a container exits unexpectedly or behaves weirdly. The logs are your first clue.

---

## `docker exec` — run a command inside a running container

**What it does:** opens a shell or runs a one-off command inside an already-running container. The container stays running; you're just attaching to it temporarily.

```powershell
docker exec <container> <command>
docker exec -it <container> bash        # interactive shell
```

> **Windows note:** use PowerShell's `docker exec` directly rather than wrapping the binary path in `sh -c`. If you use Git Bash and include paths like `/var/ossec/...`, Git Bash rewrites them to `C:/Program Files/Git/var/...` and the command fails with exit 127.

**Lab examples:**

```powershell
# Tail the Wazuh agent log on target-ubuntu
docker exec target-ubuntu sh -c "tail -20 /var/ossec/logs/ossec.log"

# List enrolled agents on the manager
docker exec docker-soc-lab-wazuh.manager-1 /var/ossec/bin/agent_control -l

# Check daemon status on the manager
docker exec docker-soc-lab-wazuh.manager-1 /var/ossec/bin/wazuh-control status

# Get an interactive shell on target-ubuntu
docker exec -it target-ubuntu bash
```

**When to use it:** inspecting files inside containers, running management tools (like `agent_control`), debugging configs. Way faster than SSHing in.

---

## `docker inspect` — get detailed container metadata

**What it does:** dumps everything Docker knows about a container — network IPs, volumes, env vars, start command, exit codes — as JSON.

```powershell
docker inspect <container>
```

Use `--format` to extract one field:

```powershell
# Get the container's IP on the docker-soc-lab_default network
docker inspect target-ubuntu --format "{{.NetworkSettings.Networks.docker-soc-lab_default.IPAddress}}"

# Get exit code of a stopped container
docker inspect target-ubuntu --format "{{.State.ExitCode}}"
```

**Lab example:** when the target-ubuntu container kept stopping, I used this to check the exit code (it was 9 = `SIGKILL`, meaning the entrypoint script died):

```powershell
docker inspect target-ubuntu --format "{{.State.Status}} (exit {{.State.ExitCode}})"
# Output: exited (exit 9)
```

**When to use it:** when you need to know a container's IP, understand its volume mounts, or diagnose unexpected exits.

---

## `docker restart` — restart a container

**What it does:** stops then starts the container, keeping all its config. The container process re-runs from the beginning.

```powershell
docker restart <container>
```

**Lab example:**
```powershell
docker restart target-ubuntu
```

**When to use it:** when a service inside a container misbehaves and you want a clean restart without recreating the container. For Wazuh daemons specifically (like `wazuh-authd`), sometimes you need to restart at the daemon level instead — see `docker exec ... wazuh-control restart`.

---

## `docker rm` — remove a stopped container

**What it does:** deletes a stopped container. The image stays; just the container instance goes.

```powershell
docker rm <container>
docker rm -f <container>    # force-stop and remove even if running
```

**Lab example:** when I needed to recreate target-ubuntu cleanly (force-recreate via compose is cleaner for this):
```powershell
docker rm -f target-ubuntu
```

**When to use it:** clearing out stopped containers, or when you need to recreate one from scratch. `docker-compose up --force-recreate` is usually the better way when using compose.

---

## `docker images` — list downloaded images

**What it does:** shows all images stored locally with their size and tag.

```powershell
docker images
```

**Lab example output:**
```
REPOSITORY                    TAG       SIZE
wazuh/wazuh-manager           4.9.0     819MB
wazuh/wazuh-indexer           4.9.0     795MB
wazuh/wazuh-dashboard         4.9.0     812MB
ubuntu                        22.04     77.9MB
wazuh/wazuh-certs-generator   0.0.2     29MB
```

**When to use it:** checking what's cached locally, checking image sizes before pulling.

---

## `docker network ls` — list Docker networks

**What it does:** lists all Docker networks on the host.

```powershell
docker network ls
```

**Lab example output:**
```
NETWORK ID     NAME                      DRIVER    SCOPE
0816988301ba   docker-soc-lab_default    bridge    local
519fa3ddd302   bridge                    bridge    local
d5afdd875b8b   host                      host      local
```

The `docker-soc-lab_default` network is the one all four containers share. Compose creates it automatically when you first run `docker-compose up`.

**When to use it:** confirming a network exists before attaching a container to it (the agent.yml compose file uses `external: true` for this network — if the main stack isn't up, the agent container will fail to start).

---

## Quick reference

| Command | What it does |
|---|---|
| `docker ps` | List running containers |
| `docker ps -a` | List all containers including stopped |
| `docker logs <name>` | Show container output |
| `docker logs -f <name>` | Follow container output live |
| `docker exec <name> <cmd>` | Run command inside container |
| `docker exec -it <name> bash` | Interactive shell inside container |
| `docker inspect <name>` | Full container metadata as JSON |
| `docker inspect <name> --format "{{...}}"` | Extract one field |
| `docker restart <name>` | Restart a container |
| `docker rm -f <name>` | Force-remove a container |
| `docker images` | List downloaded images |
| `docker network ls` | List Docker networks |
