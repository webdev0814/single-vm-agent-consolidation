# Skill: Consolidate Hermes Agent Framework onto an Existing OpenClaw Instance

**Skill ID:** `hermes-openclaw-consolidation`  
**Category:** Infrastructure / Agent Management  
**Frameworks:** Hermes Agent (Python/uv), OpenClaw (Node.js)  
**Platform:** Oracle Cloud Infrastructure (OCI) — ARM A1.Flex  
**Difficulty:** Advanced  
**Estimated time:** 3–6 hours depending on blockers

---

## Overview

This skill describes how to migrate a running Hermes agent installation from a dedicated paid VM onto an existing OpenClaw VM, consolidating both frameworks onto a single Oracle Always Free tier instance. The result eliminates the second VM's charges while keeping all agent profiles, memory, skills, session data, and Telegram connections intact.

This skill was developed and validated in a live production migration. The troubleshooting sections document real failure modes encountered during that migration.

---

## When to use this skill

Use this skill when:
- You are running Hermes agents and OpenClaw agents on separate OCI VMs
- The combined OCPU count exceeds the Always Free limit (4 OCPUs / 24GB RAM), causing unexpected charges
- You want to consolidate to a single VM without losing agent state
- You need to migrate Hermes profiles (including memory, SOUL, skills, auth) to a new host

---

## Prerequisites

### Source machine (Hermes VM)
- Ubuntu 22.04+ ARM64
- Hermes Agent installed at `~/.hermes/hermes-agent/`
- Agent profiles at `~/.hermes/profiles/{agent-name}/`
- OCI CLI configured with valid credentials (`~/.oci/config`)
- SSH access

### Destination machine (OpenClaw VM)
- Ubuntu 22.04+ ARM64
- OpenClaw installed and running
- Python 3.11+ available or installable via `uv`
- Minimum 20GB free disk space
- SSH access

### Your workstation
- SSH access to both machines
- OCI console access
- Telegram access to test agents after migration

---

## Architecture after migration

```
Single OCI VM (4 OCPU / 24GB RAM — Always Free)
├── OpenClaw (Node.js) — existing agents unchanged
│   └── Port 18789 (gateway)
└── Hermes Agent (Python/uv) — migrated from second VM
    ├── Port 9080 (Whisper STT)
    ├── Port 9119 (Hermes dashboard, optional)
    └── Agent gateways (configurable ports)
```

Both frameworks run independently with no shared state. Port conflicts are the only integration concern.

---

## Phase 1 — Pre-flight on the Hermes (source) VM

### 1.1 — Document running services

```bash
systemctl list-units --type=service --state=running
ps aux --sort=-%mem | head -40
ss -tlnp
```

Record all running services and listening ports. This is your migration baseline.

### 1.2 — Create a backup archive

Exclude the Python venv (large and reproducible) but include all profile data:

```bash
tar -czf /tmp/hermes_backup_$(date +%Y%m%d_%H%M%S).tar.gz \
  --exclude='/home/ubuntu/.hermes/hermes-agent/.venv' \
  --exclude='/home/ubuntu/.hermes/hermes-agent/venv' \
  --exclude='/home/ubuntu/.cache' \
  --exclude='/home/ubuntu/snap' \
  --warning=no-file-changed \
  /home/ubuntu/.hermes \
  /home/ubuntu/vertex_proxy.py \
  /etc/nginx \
  /etc/systemd/system/whisper-stt.service \
  /etc/systemd/system/gemini-proxy.service \
  2>/tmp/backup_errors.log

# Verify
gzip -t /tmp/hermes_backup_*.tar.gz && echo "ARCHIVE OK" || echo "CORRUPT"
sha256sum /tmp/hermes_backup_*.tar.gz
cat /tmp/backup_errors.log
```

**Important:** Verify the archive contains session and memory files:

```bash
tar -tzf /tmp/hermes_backup_*.tar.gz | \
  grep -E "(\.session|MEMORY|SOUL|\.env|nginx)" | head -30
```

Post the checksum. Do not proceed until the archive is verified OK.

> **Troubleshooting — archive grows to unexpected size:**  
> The venv directory may not be excluded if it lacks a dot prefix. Check the actual venv path: `ls ~/.hermes/hermes-agent/` and adjust the exclude pattern to match the exact directory name (`venv` vs `.venv`).

> **Troubleshooting — `file changed as we read it` errors on critical files:**  
> Stop the agent gateway services temporarily during the backup:  
> `sudo systemctl stop hermes-michael hermes-dwight hermes-kevin`  
> Then run the backup and restart immediately after.

### 1.3 — Transfer the archive

On the destination machine, ensure the source machine's public key is in `~/.ssh/authorized_keys`, then pull from destination:

```bash
# On destination machine
mkdir -p ~/hermes_migration
rsync -avP --checksum \
  -e "ssh -i ~/.ssh/YOUR_KEY" \
  ubuntu@SOURCE_HOST:/tmp/hermes_backup_*.tar.gz \
  ~/hermes_migration/

sha256sum ~/hermes_migration/hermes_backup_*.tar.gz
```

Checksums must match exactly before proceeding.

> **Troubleshooting — SSH permission denied:**  
> The key used to pull must be the private key whose public key is in the source machine's `~/.ssh/authorized_keys`. Run `ssh-keygen -l -f ~/.ssh/YOUR_KEY.pub` on both sides to compare fingerprints and confirm the pair matches.

### 1.4 — Take an OCI boot volume snapshot

```bash
# Get boot volume OCID
oci compute boot-volume-attachment list \
  --compartment-id COMPARTMENT_OCID \
  --instance-id HERMES_INSTANCE_OCID \
  --query "data[0].\"boot-volume-id\"" \
  --raw-output

# Create snapshot
oci bv boot-volume-backup create \
  --boot-volume-id BOOT_VOLUME_OCID \
  --display-name "hermes-premigration-$(date +%Y%m%d)" \
  --type INCREMENTAL

# Wait for AVAILABLE
oci bv boot-volume-backup list \
  --compartment-id COMPARTMENT_OCID \
  --query "data[?\"lifecycle-state\"=='AVAILABLE'].{name:\"display-name\",state:\"lifecycle-state\"}"
```

### 1.5 — Audit hardcoded IP references

The source VM's public IP will be released on termination. Find anything that references it:

```bash
SOURCE_IP=$(curl -s ifconfig.me)
grep -rn "$SOURCE_IP" \
  /home/ /etc/ /opt/ \
  --include="*.conf" --include="*.env" \
  --include="*.json" --include="*.yaml" \
  --include="*.py" --include="*.sh" \
  2>/dev/null
```

Update any hardcoded references to point to the destination VM's IP before proceeding.

> **Note on Tailscale:** If you use Tailscale, it is NOT affected by IP changes. Tailscale communication uses `100.x.x.x` mesh addresses tied to device identity, not the underlying public IP.

---

## Phase 2 — Prepare the destination (OpenClaw) VM

### 2.1 — Audit current state

```bash
df -h && lsblk
ss -tlnp
systemctl list-units --type=service --state=running
cat /etc/nginx/sites-enabled/* 2>/dev/null
```

Record all listening ports. Compare against Hermes ports from Phase 1.1.

**Common Hermes ports to check for conflicts:**
- `9080` — Whisper STT
- `9119` — Hermes dashboard
- `8644` — Agent gateway (may vary per profile)
- `4001` — vertex_proxy (may already exist)
- `80/443` — nginx (will need config merge, not replacement)

### 2.2 — Expand boot volume if needed

Always Free tier provides 200GB total block storage. If the destination VM needs more space:

```bash
# Step A: Resize at OCI level
oci bv boot-volume update \
  --boot-volume-id DEST_BOOT_VOLUME_OCID \
  --size-in-gbs 150

# Step B: Extend partition and filesystem inside OS (REQUIRED — OCI resize alone is not enough)
sudo growpart /dev/sda 1
sudo resize2fs /dev/sda1
df -h /  # Must show new size
```

> **Critical:** Both steps are required. If `df -h` still shows the old size after the OCI resize, the OS partition has not been extended. Run `growpart` and `resize2fs` to claim the space.

### 2.3 — Install Python 3.11+ via uv

Hermes requires Python 3.11+. The deadsnakes PPA may not have ARM64 packages for Ubuntu 20.04. Use `uv` instead (same tool Hermes uses internally):

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc
uv python install 3.11
uv python list  # confirm 3.11 available
```

### 2.4 — Extract and install Hermes framework

```bash
mkdir -p /tmp/hermes_extracted

# Extract framework code only (not profiles yet)
tar -xzf ~/hermes_migration/hermes_backup_*.tar.gz \
  --wildcards 'home/ubuntu/.hermes/hermes-agent/*' \
  --exclude='home/ubuntu/.hermes/hermes-agent/venv' \
  --exclude='home/ubuntu/.hermes/hermes-agent/.venv' \
  -C /tmp/hermes_extracted/

# Copy to correct location
mkdir -p ~/.hermes/hermes-agent
cp -r /tmp/hermes_extracted/home/ubuntu/.hermes/hermes-agent/* \
  ~/.hermes/hermes-agent/

# Create venv and install using uv
cd ~/.hermes/hermes-agent
uv venv --python 3.11 venv
source venv/bin/activate
uv pip install -e . || uv sync

# Verify
hermes --version
```

---

## Phase 3 — Restore agent profiles

### 3.1 — Extract profiles from archive

```bash
mkdir -p /tmp/profile_restore

tar -xzf ~/hermes_migration/hermes_backup_*.tar.gz \
  --wildcards \
  'home/ubuntu/.hermes/profiles/*' \
  'home/ubuntu/.hermes/memories/*' \
  'home/ubuntu/.hermes/SOUL.md' \
  'home/ubuntu/.hermes/.env' \
  'home/ubuntu/.hermes/config.yaml' \
  'home/ubuntu/.hermes/auth.json' \
  -C /tmp/profile_restore/

# Verify key files present
find /tmp/profile_restore -maxdepth 5 | grep -E "(MEMORY|SOUL|\.env|auth)" | head -20
```

### 3.2 — Copy profiles to destination

```bash
# Copy profiles
cp -r /tmp/profile_restore/home/ubuntu/.hermes/profiles ~/.hermes/
cp -r /tmp/profile_restore/home/ubuntu/.hermes/memories ~/.hermes/
cp /tmp/profile_restore/home/ubuntu/.hermes/.env ~/.hermes/
cp /tmp/profile_restore/home/ubuntu/.hermes/SOUL.md ~/.hermes/
cp /tmp/profile_restore/home/ubuntu/.hermes/config.yaml ~/.hermes/
cp /tmp/profile_restore/home/ubuntu/.hermes/auth.json ~/.hermes/

# Fix ownership
sudo chown -R ubuntu:ubuntu ~/.hermes/

# Verify
find ~/.hermes/profiles -name "MEMORY.md" -exec ls -lah {} \;
find ~/.hermes/profiles -name "SOUL.md" -exec ls -lah {} \;
```

---

## Phase 4 — Handle authentication (openai-codex provider)

This is the most complex phase. Hermes agents that use the `openai-codex` provider authenticate via OAuth tokens stored in `~/.hermes/auth.json` and per-profile `auth.json` files. These tokens do not transfer cleanly and require re-authentication on the new machine.

### 4.1 — Check for auth errors

```bash
python3 -c "
import json
for profile in ['michael', 'dwight', 'kevin']:
    try:
        with open(f'$HOME/.hermes/profiles/{profile}/auth.json') as f:
            d = json.load(f)
        err = d.get('providers', {}).get('openai-codex', {}).get('last_auth_error')
        print(f'{profile}: {err}')
    except: print(f'{profile}: no auth.json')
"
```

### 4.2 — Re-authenticate openai-codex

If any profile shows `refresh_token_reused` or `relogin_required: true`:

```bash
# Install codex CLI if not present (find it via OpenClaw if available)
CODEX=$(find ~/.openclaw -name "codex" -type f 2>/dev/null | head -1)

# Run login — will print a browser URL
$CODEX login 2>&1 | tee /tmp/codex_login.log &
sleep 10
cat /tmp/codex_login.log
```

Open the URL in a browser logged into your OpenAI account. After completing login, the fresh tokens are stored in `~/.codex/auth.json`.

### 4.3 — Transplant fresh tokens into Hermes

```bash
python3 << 'EOF'
import json, shutil
from datetime import datetime, timezone

with open('/home/ubuntu/.codex/auth.json') as f:
    codex = json.load(f)

fresh_tokens = codex['tokens']
last_refresh = codex.get('last_refresh', datetime.now(timezone.utc).isoformat())

# Update top-level Hermes auth
with open('/home/ubuntu/.hermes/auth.json') as f:
    hermes = json.load(f)

hermes['providers']['openai-codex']['tokens'] = fresh_tokens
hermes['providers']['openai-codex']['last_refresh'] = last_refresh
hermes['updated_at'] = datetime.now(timezone.utc).isoformat()

with open('/home/ubuntu/.hermes/auth.json', 'w') as f:
    json.dump(hermes, f, indent=2)
print("Top-level auth.json updated")
EOF
```

### 4.4 — Fix per-profile credential pools

Each profile maintains its own `credential_pool` for openai-codex. Duplicate or stale entries cause silent crashes. Collapse each profile's pool to a single clean entry matching the top-level:

```bash
python3 << 'EOF'
import json, shutil
from datetime import datetime, timezone

with open('/home/ubuntu/.hermes/auth.json') as f:
    top = json.load(f)

fresh_entry = top['credential_pool']['openai-codex'][0].copy()
# Clear all error fields
for field in ['last_error_code','last_error_reason','last_error_message',
              'last_error_reset_at','last_status','last_status_at']:
    fresh_entry[field] = None

for profile in ['michael', 'dwight', 'kevin']:  # adjust to your profile names
    path = f'/home/ubuntu/.hermes/profiles/{profile}/auth.json'
    try:
        shutil.copy(path, path + f'.bak-{datetime.now().strftime("%Y%m%d_%H%M%S")}')
        with open(path) as f:
            d = json.load(f)

        d['credential_pool']['openai-codex'] = [fresh_entry]
        if 'openai-codex' in d.get('providers', {}):
            d['providers']['openai-codex'].pop('last_auth_error', None)
            d['providers']['openai-codex']['last_refresh'] = fresh_entry.get('last_refresh','')
        d['updated_at'] = datetime.now(timezone.utc).isoformat()

        with open(path, 'w') as f:
            json.dump(d, f, indent=2)
        print(f"{profile}: credential pool fixed")
    except Exception as e:
        print(f"{profile}: {e}")
EOF
```

---

## Phase 5 — Start services and test

### 5.1 — Stop Hermes services on source VM first

Two instances of the same Telegram bot token cannot poll simultaneously. Stop the source VM's Hermes services before starting on the destination:

```bash
# On source VM — disable systemd services so they don't auto-restart
sudo systemctl stop hermes-michael hermes-dwight hermes-kevin whisper-stt
sudo systemctl disable hermes-michael hermes-dwight hermes-kevin whisper-stt
```

Verify they are stopped and will not respawn:
```bash
systemctl is-active hermes-michael hermes-dwight hermes-kevin
# All should show: inactive
```

### 5.2 — Start services on destination VM

Start one at a time and test each before starting the next:

```bash
cd ~/.hermes/hermes-agent
source venv/bin/activate

# Start Whisper first (no auth state, lowest risk)
uvicorn whisper_server:app --host 0.0.0.0 --port 9080 &
sleep 5 && ss -tlnp | grep 9080

# Start agents one at a time — test each on Telegram before proceeding
python -m hermes_cli.main --profile kevin gateway run --replace \
  > ~/.hermes/profiles/kevin/logs/gateway.log 2>&1 &
```

Test on Telegram, then repeat for each remaining profile.

> **Troubleshooting — agent starts then exits silently (0-byte log):**  
> This is almost always a credential pool issue. Run Phase 4.4 again for the affected profile. The symptom is the process lives for 20-40 seconds then exits without writing any output — this happens because the crash occurs before Hermes logging initializes.

> **Troubleshooting — Telegram polling conflict:**  
> `Error: Conflict: terminated by other getUpdates request`  
> The source VM's services are still running. SSH into the source VM and confirm all Hermes gateway processes are dead: `ps aux | grep hermes_cli | grep -v grep` — if any appear, kill them with `pkill -f hermes_cli.main`.

> **Troubleshooting — agent runs but never binds a port:**  
> This is normal for some Hermes configurations. The agent communicates via outbound Telegram connections, not inbound. Test by sending a Telegram message — if the agent responds, it is working correctly regardless of whether a local port is bound.

### 5.3 — Create systemd service units for persistence

```bash
for PROFILE in michael dwight kevin; do
sudo tee /etc/systemd/system/hermes-${PROFILE}.service > /dev/null << EOF
[Unit]
Description=Hermes Agent - ${PROFILE}
After=network.target
StartLimitIntervalSec=60
StartLimitBurst=3

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/.hermes/hermes-agent
ExecStart=/home/ubuntu/.hermes/hermes-agent/venv/bin/python -m hermes_cli.main --profile ${PROFILE} gateway run --replace
Restart=on-failure
RestartSec=10
StandardOutput=append:/home/ubuntu/.hermes/profiles/${PROFILE}/logs/gateway.log
StandardError=append:/home/ubuntu/.hermes/profiles/${PROFILE}/logs/errors.log

[Install]
WantedBy=multi-user.target
EOF
done

sudo tee /etc/systemd/system/hermes-whisper.service > /dev/null << 'EOF'
[Unit]
Description=Hermes Whisper STT Server
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/.hermes/hermes-agent
ExecStart=/home/ubuntu/.hermes/hermes-agent/venv/bin/python3 /home/ubuntu/.hermes/hermes-agent/venv/bin/uvicorn whisper_server:app --host 0.0.0.0 --port 9080
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable hermes-michael hermes-dwight hermes-kevin hermes-whisper
```

---

## Phase 6 — Terminate source VM and clean up OCI

Only proceed after all agents are confirmed responding on Telegram with systemd persistence enabled.

### 6.1 — Terminate the source VM

```bash
oci compute instance terminate \
  --instance-id SOURCE_INSTANCE_OCID \
  --preserve-boot-volume false \
  --force

# Wait for TERMINATED
oci compute instance get \
  --instance-id SOURCE_INSTANCE_OCID \
  --query 'data."lifecycle-state"'
```

### 6.2 — Delete networking resources in source compartment

Delete in dependency order:

```bash
COMP_ID="SOURCE_COMPARTMENT_OCID"

# List everything first
oci network vcn list --compartment-id $COMP_ID
oci network subnet list --compartment-id $COMP_ID
oci network internet-gateway list --compartment-id $COMP_ID

# Delete: security rules → route tables → subnet → internet gateway → VCN
```

### 6.3 — Delete the compartment

```bash
oci iam compartment delete \
  --compartment-id SOURCE_COMPARTMENT_OCID \
  --force
```

> **Troubleshooting — compartment reverts from DELETING to ACTIVE:**  
> OCI requires all resources to be in a fully purged state before compartment deletion succeeds. TERMINATED instances can take 10-30 minutes to fully purge from OCI's backend. Retry the delete command every 5 minutes. Also check for boot volume backups — even TERMINATED backups can block deletion:  
> `oci bv boot-volume-backup list --compartment-id COMP_ID --query "data[*].{name:\"display-name\",state:\"lifecycle-state\"}"`  
> Delete any that appear, then retry compartment deletion.

### 6.4 — Set a budget alert to prevent future surprise charges

```bash
# Create a $1 budget (minimum OCI allows)
oci budgets budget create \
  --compartment-id ROOT_COMPARTMENT_OCID \
  --display-name "zero-cost-alert" \
  --amount 1 \
  --reset-period MONTHLY \
  --budget-processing-period-start-offset 1 \
  --targets '[{"targetType":"COMPARTMENT","values":["ROOT_COMPARTMENT_OCID"]}]'

# Set alert rule at $0.01 so any charge triggers notification
oci budgets alert-rule create \
  --budget-id BUDGET_OCID \
  --display-name "any-spend-alert" \
  --threshold 0.01 \
  --threshold-type ABSOLUTE \
  --type ACTUAL \
  --recipients "YOUR_EMAIL@example.com" \
  --message "OCI charge detected. Investigate immediately."
```

---

## Verification checklist

Run these checks after migration is complete:

- [ ] All agents responding on Telegram with correct memory/persona
- [ ] `systemctl status hermes-michael hermes-dwight hermes-kevin hermes-whisper` — all active
- [ ] Source VM shows TERMINATED in OCI console
- [ ] Source compartment shows DELETED in OCI console
- [ ] Only destination VM (4 OCPU / 24GB) visible under Compute
- [ ] Budget alert active in OCI console
- [ ] `df -h` on destination shows healthy disk usage
- [ ] `free -h` on destination shows sufficient RAM headroom

---

## Free tier math reference

Oracle Always Free ARM A1.Flex limits (as of 2026):

| Resource | Free limit | Safe single-VM config |
|---|---|---|
| OCPU-hours/month | 3,000 | 4 OCPUs × 730h = 2,920 ✅ |
| GB-RAM-hours/month | 18,000 | 24GB × 730h = 17,520 ✅ |
| Block storage | 200GB total | 150GB boot volume ✅ |

Running two VMs simultaneously (e.g. 4+2 = 6 OCPUs) exceeds the OCPU-hour limit and triggers pay-as-you-go billing for the entire compute usage, not just the overage.

---

## Key lessons learned

**1. Agents must not provision infrastructure autonomously.**  
Any agent with OCI CLI credentials can create VMs, compartments, and networking resources. This is what caused the original cost overrun — an agent provisioned a second VM without human approval. Always require explicit human confirmation before any infrastructure provisioning action.

**2. The venv is always excludable.**  
The Python virtual environment is large (1-2GB) and fully reproducible from `pyproject.toml` + `uv.lock`. Never include it in migration archives.

**3. openai-codex tokens do not survive machine transfers.**  
The OAuth token refresh mechanism detects reuse across different clients and invalidates the refresh token. Always plan for a re-authentication step when migrating Hermes to a new machine that uses the openai-codex provider.

**4. Silent crashes before logging initializes = credential pool issue.**  
If a Hermes agent process starts, runs for 20-40 seconds, and exits without writing any logs, the crash is happening before Hermes's logging system initializes. This is almost always caused by a stale or duplicate entry in the profile's `credential_pool` for the configured LLM provider.

**5. OCI compartment deletion is asynchronous and can take 30+ minutes.**  
TERMINATED resources are not immediately purged. Do not assume compartment deletion will succeed immediately after instance termination. Budget up to an hour and retry periodically.

**6. Telegram bot tokens are single-session.**  
Two processes sharing the same bot token cannot both poll Telegram simultaneously. Always stop the source VM's Hermes services before starting them on the destination, and verify the source services are fully stopped and cannot auto-restart via systemd.

---

## Files and paths reference

| Path | Description |
|---|---|
| `~/.hermes/` | Hermes root directory |
| `~/.hermes/hermes-agent/` | Framework installation |
| `~/.hermes/hermes-agent/venv/` | Python virtual environment |
| `~/.hermes/profiles/{name}/` | Per-agent profile data |
| `~/.hermes/profiles/{name}/MEMORY.md` | Agent long-term memory |
| `~/.hermes/profiles/{name}/SOUL.md` | Agent identity/persona |
| `~/.hermes/profiles/{name}/.env` | Agent environment variables |
| `~/.hermes/profiles/{name}/auth.json` | Agent provider credentials |
| `~/.hermes/profiles/{name}/state.db` | Agent conversation history (SQLite) |
| `~/.hermes/auth.json` | Top-level provider credentials |
| `~/.hermes/config.yaml` | Top-level Hermes configuration |
| `~/.codex/auth.json` | Codex CLI auth store (fresh tokens after login) |
| `~/.openclaw/` | OpenClaw root directory |

---

*Skill developed from live production migration, June 2026.*  
*Validated on: Ubuntu 22.04 ARM64, OCI A1.Flex, Hermes Agent v0.15.1, OpenClaw (Node.js/v24)*
