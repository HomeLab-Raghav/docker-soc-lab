# 06 — Troubleshooting

Real issues I hit while building this lab, what caused each one, and exactly how I fixed it.

---

## Issue 1: Indexer exits immediately / container won't start

**Symptom:** `docker-soc-lab-wazuh.indexer-1` shows `Exited (137)` in `docker ps -a`. The logs show:

```
max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```

**Cause:** OpenSearch (which the Wazuh indexer is built on) requires the Linux kernel to support a much higher number of virtual memory mappings than the default. On Windows, Docker runs inside a WSL2 VM — that VM has the low default limit.

**Fix:**

```powershell
wsl -d docker-desktop sysctl -w vm.max_map_count=262144
```

Then restart the indexer:

```powershell
docker-compose up -d
```

**Important:** this setting resets every time Docker Desktop restarts. If the indexer dies after a reboot, this is almost certainly why. Run the `wsl` command again and it'll come back up.

---

## Issue 2: `docker-compose -f agent.yml up -d` fails with "network not found"

**Symptom:**

```
Error response from daemon: network docker-soc-lab_default not found
```

**Cause:** `agent.yml` uses `external: true` for the `docker-soc-lab_default` network, meaning it expects that network to already exist. The network is created by the main `docker-compose up -d`. If the main stack isn't running, the network doesn't exist.

**Fix:** start the main stack first:

```powershell
docker-compose up -d        # this creates docker-soc-lab_default
docker-compose -f agent.yml up -d
```

---

## Issue 3: `ERROR: Duplicate agent name: target-ubuntu (from manager)` — agent won't enroll

This was the most involved issue I hit. The agent log repeated:

```
ERROR: Duplicate agent name: target-ubuntu (from manager)
ERROR: Unable to add agent (from manager)
```

**Root cause (three layers):**

**Layer 1:** All Wazuh manager daemons were stopped after a container restart. `wazuh-authd` (port 1515) wasn't running, so enrollment requests either bounced or were partially processed.

**Layer 2:** The real trap: `wazuh-db` (the manager's database daemon) holds the definitive record of enrolled agents. When you remove an agent with `manage_agents -r`, it has to update both the flat `client.keys` file *and* the `wazuh-db` database. If `wazuh-db` isn't running at the time of removal, only the file gets updated — the database record survives, and `authd` repopulates `client.keys` from the DB on its next start.

Clearing `client.keys` directly (truncating the file) is useless — authd overwrites it from the database within seconds.

**Layer 3:** A bug in `agent.yml`'s entrypoint: `useradd victim` is chained with `&&`, so on `docker restart` (where the container FS persists), `useradd` exits 9 ("user already exists") and kills the chain before `sshd -D` — the container dies immediately.

**Fix — step by step:**

**Step 1:** Start the manager's daemons (including `wazuh-db`):

```powershell
docker exec docker-soc-lab-wazuh.manager-1 /var/ossec/bin/wazuh-control start
```

**Step 2:** Remove the stale agent entry *with the DB running* (answer `y` when prompted):

```powershell
"y" | docker exec -i docker-soc-lab-wazuh.manager-1 /var/ossec/bin/manage_agents -r <ID>
```

Where `<ID>` is the stale agent's ID from `manage_agents -l`. The output should say `Agent 'XXX' removed.` with no "Could not remove the DB" error.

**Step 3:** Restart `authd` to flush its in-memory copy:

```powershell
docker exec docker-soc-lab-wazuh.manager-1 /var/ossec/bin/wazuh-control restart
```

**Step 4:** Fix `agent.yml` to survive restarts (make user creation idempotent):

```yaml
# Before (breaks on restart)
/var/ossec/bin/wazuh-control start &&
useradd -m -s /bin/bash victim &&

# After (works on restart)
/var/ossec/bin/wazuh-control restart &&
(id victim || useradd -m -s /bin/bash victim) &&
```

**Step 5:** Recreate target-ubuntu cleanly:

```powershell
docker-compose -f agent.yml up -d --force-recreate
```

**Verify:**

```powershell
# Agent log should show "Connected to the server"
docker exec target-ubuntu sh -c "tail -20 /var/ossec/logs/ossec.log"

# Manager should show target-ubuntu as Active
docker exec docker-soc-lab-wazuh.manager-1 /var/ossec/bin/agent_control -l
```

**Key lesson:** `client.keys` is a derived file. The source of truth is `wazuh-db`. Never try to fix enrollment issues by editing `client.keys` directly — always use `manage_agents` while `wazuh-db` is running.

---

## Issue 4: `manage_agents` path error in Git Bash

**Symptom:**

```
OCI runtime exec failed: exec failed: unable to start container process:
exec: "C:/Program Files/Git/var/ossec/bin/manage_agents": no such file or directory
```

**Cause:** Git Bash on Windows rewrites paths that start with `/` to `C:/Program Files/Git/...`. So when you run:

```bash
docker exec container /var/ossec/bin/manage_agents
```

Git Bash sees `/var/...` and silently transforms it to `C:/Program Files/Git/var/...`, which Docker then passes to the container as the executable path — which doesn't exist.

**Fix:** Use PowerShell instead of Git Bash for any `docker exec` commands that reference absolute Linux paths:

```powershell
# This works in PowerShell — no path mangling
docker exec docker-soc-lab-wazuh.manager-1 /var/ossec/bin/agent_control -l
```

Or wrap in `sh -c "..."` (but only for paths that don't start the command itself):

```bash
# Also works in Git Bash — sh -c prevents the path rewrite
docker exec container sh -c '/var/ossec/bin/agent_control -l'
```

---

## Issue 5: target-ubuntu container exits immediately on `docker restart`

**Symptom:** `docker ps` shows target-ubuntu as `Exited (9)` seconds after a restart.

**Cause:** As described in Issue 3, the entrypoint's `&&` chain reaches `useradd victim`, which exits non-zero because the user already exists (the container FS persists across restarts), killing the whole chain before `sshd -D`.

**Fix:** Make user creation idempotent — see Issue 3, Step 4 above.

---

## Quick diagnostic checklist

When something in the lab is broken, run these in order:

```powershell
# 1. Are all containers actually running?
docker ps --format "table {{.Names}}\t{{.Status}}"

# 2. If not, what do the logs say?
docker logs <container-name> --tail 30

# 3. Are the manager daemons running?
docker exec docker-soc-lab-wazuh.manager-1 /var/ossec/bin/wazuh-control status

# 4. Is the agent connected?
docker exec target-ubuntu sh -c "tail -20 /var/ossec/logs/ossec.log"

# 5. Does the manager see the agent as Active?
docker exec docker-soc-lab-wazuh.manager-1 /var/ossec/bin/agent_control -l

# 6. Does the shared network exist?
docker network ls | findstr soc-lab
```
