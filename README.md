# hermes-openclaw-consolidation

> **Stop paying for two VMs. Merge your Hermes Agent and OpenClaw onto a single Oracle Always Free instance — zero downtime, zero lost memory.**

---

## What this does

This is a battle-tested migration guide for consolidating a [Hermes Agent](https://github.com/NousResearch/hermes-agent) installation from a dedicated OCI VM onto an existing [OpenClaw](https://openclaw.ai) VM. It eliminates the second VM's charges while keeping **every agent profile, memory, persona, skill, session history, and Telegram connection** intact.

Written from a **live production migration** — every troubleshooting section documents a real failure mode that was encountered and resolved.

## Who this is for

- You run Hermes agents (Python/uv) and OpenClaw agents (Node.js) on **separate OCI ARM A1.Flex VMs**
- Your combined OCPU count exceeds the Always Free limit (4 OCPUs / 24GB RAM), triggering **unexpected charges**
- You want to consolidate to a single VM without losing any agent state
- You need to migrate Hermes profiles (memory, SOUL, skills, auth) to a new host

## What's covered

| Phase | What happens |
|-------|-------------|
| **Phase 1** | Pre-flight: document services, create verified backup, OCI snapshot, audit hardcoded IPs |
| **Phase 2** | Prepare destination: expand boot volume, install Python via `uv`, extract framework |
| **Phase 3** | Restore agent profiles: memories, personas, skills, configs |
| **Phase 4** | Handle authentication: re-auth openai-codex, fix credential pools (the hardest part) |
| **Phase 5** | Start services: avoid Telegram polling conflicts, systemd persistence |
| **Phase 6** | Terminate source VM, delete OCI resources, set budget alerts |

## Key lessons baked in

1. **Agents must not provision infrastructure autonomously** — an agent created a second VM without approval, causing the original cost overrun
2. **OAuth tokens don't survive machine transfers** — plan for re-authentication
3. **Silent crashes before logging = credential pool issue** — the exact diagnostic is documented
4. **OCI compartment deletion takes 30+ minutes** — don't assume instant cleanup
5. **Telegram bot tokens are single-session** — two processes can't poll simultaneously

## Prerequisites

- Two OCI ARM A1.Flex VMs (source + destination)
- Ubuntu 22.04+ ARM64 on both
- SSH access to both machines
- OCI CLI configured
- Telegram access for post-migration testing

## Quick start

```bash
# 1. On the source VM — create the backup
bash backup.sh

# 2. Transfer to destination
rsync -avP /tmp/hermes_backup_*.tar.gz ubuntu@DEST_HOST:~/hermes_migration/

# 3. Follow the phases in SKILL.md on the destination
```

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

## Free tier math

| Resource | Free limit | Single-VM config | Status |
|----------|-----------|-----------------|--------|
| OCPU-hours/month | 3,000 | 4 × 730h = 2,920 | ✅ Under limit |
| GB-RAM-hours/month | 18,000 | 24GB × 730h = 17,520 | ✅ Under limit |
| Block storage | 200GB | 150GB boot volume | ✅ Plenty of room |

Two VMs (4+2 = 6 OCPUs) exceeds the limit and triggers pay-as-you-go for **all** compute — not just the overage.

## License

MIT — use it, fork it, save yourself some money.

---

*Developed from a live production migration, June 2026. Validated on Ubuntu 22.04 ARM64, OCI A1.Flex, Hermes Agent v0.15.1, OpenClaw (Node.js/v24).*
