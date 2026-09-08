# Agent Briefing: single-vm-agent-consolidation

## 1. Repository Overview & Purpose
- **Repository**: `webdev0814/single-vm-agent-consolidation`
- **Visibility**: `Public`
- **Default Branch**: `main`
- **Last Updated / Pushed**: 2026-09-08
- **Description**: Battle-tested guide to merge Hermes Agent and OpenClaw onto a single Oracle Always Free VM with zero downtime.
- **Context from README**: > **Stop paying for two VMs. Merge your Hermes Agent and OpenClaw onto a single Oracle Always Free instance — zero downtime, zero lost memory.** --- This is a battle-tested migration guide for consolidating a [Hermes Agent](https://github.com/NousResearch/hermes-agent) installation from a dedicated ...
- **Topics/Tags**: docker, hermes, oracle-cloud, vm-migration

---

## 2. Tech Stack & Architecture
- **Primary Language / Ecosystem**: General / Multi-language
- **Key Directories**: Single root directory structure.
- **Notable Top-Level Files**: `.gitignore`, `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `LICENSE`, `README.md`, `SKILL.md`

---

## 3. Setup & Execution Commands
### Environment Setup & Installation
```bash
# Review repository files and install dependencies corresponding to the language/runtime.
```

### Running / Starting
```bash
# Check main entry point scripts or config files.
```

### Testing / Verification
```bash
# Run relevant unit/integration tests (e.g. pytest or npm test)
```

---

## 4. Recent Commit Activity (Where We Left Off)
The most recent commits show the latest development trajectory:
- `[eaa9c6a]` (2026-09-08) docs: update agent briefing with multi-computer handoff protocol
- `[d3111b2]` (2026-09-08) docs: update agent briefing with multi-computer handoff protocol
- `[43181cb]` (2026-09-08) docs: update agent briefing with multi-computer handoff protocol
- `[fd86fb4]` (2026-09-08) docs: update agent briefing with multi-computer handoff protocol
- `[f5038ac]` (2026-09-08) docs: update agent briefing with multi-computer handoff protocol
- `[f769eeb]` (2026-09-08) docs: update agent briefing with multi-computer handoff protocol
- `[09016e7]` (2026-09-08) docs: update agent briefing with multi-computer handoff protocol
- `[1e9a3a1]` (2026-09-08) docs: update agent briefing with multi-computer handoff protocol
- `[4bf80c7]` (2026-09-08) docs: update agent briefing with multi-computer handoff protocol
- `[32e66e5]` (2026-09-08) docs: update agent briefing (AGENTS.md, GEMINI.md, CLAUDE.md)

---

## 5. Current State & Immediate Next Steps
- **Current State**: Project is active under branch `main`.
- **When picking up this repo**:
  1. Inspect the top-level files and recent commits to understand the active feature or bugfix context.
  2. Verify all required credentials and environment variables before running integration scripts.
  3. Ensure all tests and linting pass after making modifications.
  4. Follow the repository conventions and preserve existing architecture patterns.

---

## 6. Multi-Computer Handoff & Git Sync Protocol
- **On Session Start**: Always run `git pull` when opening this repository on any computer to synchronize the latest changes.
- **On Task Completion**: Before ending any agent session, the agent **MUST**:
  1. Update Section 5 (Current State & Next Steps) in this `AGENTS.md` file.
  2. Stage all modifications (`git add .`).
  3. Commit with a concise conventional message (`git commit -m "feat/fix: ..."`).
  4. Push directly to GitHub (`git push`).
- **Secret Hygiene**: NEVER commit plain-text API keys, tokens, or credentials into repository files.
