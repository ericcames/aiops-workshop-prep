# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Docs
- `README.md` — move Splunk API token collection into `/aiops-setup` skill; simplify module map to a single note referencing issue #1
- `docs/dev-environment.md.example` — clarify Splunk API Token field is collected by `/aiops-setup` during credential collection
- `CLAUDE.md` — add issue #1 to open issues table (replaces closed duplicate #13)
- `.claude/commands/aiops-setup.md`, `aiops-preflight.md`, `aiops-reset.md` — add Splunk API token to credential collection prompt and `docs/dev-environment.md` template

### Docs (issue #7)
- `images/rhdp-catalog-item.png` — screenshot of the RHDP catalog item embedded in README
- `README.md` — Getting Started section simplified to 3 steps (order → clone + `claude .` → `/aiops-setup`); Claude handles credential collection automatically; correct showroom section names to Part 1/2/3
- `.claude/commands/aiops-preflight.md`, `aiops-setup.md`, `aiops-reset.md` — if `docs/dev-environment.md` is missing, ask the user for RHDP credentials and create the file automatically instead of stopping with an error
- `docs/dev-environment.md.example` — template with all required fields and placeholder values
- `docs/troubleshooting.md` — 10 common failure modes with diagnosis and fix steps
- `README.md` — add RHDP catalog item name, workshop module map, `dev-environment.md.example` setup step, link to troubleshooting guide
- `CLAUDE.md` — add open issues table, RHDP catalog item, Claude Code skills section; fix Phase 2 reset status (now tested); document Paramiko ProxyCommand limitation

### Fixed
- `reset_phase2_network.yml` — replace `ansible.netcommon.cli_config` (Paramiko fails to handle ProxyCommand and cannot resolve internal hostname `cisco-rtr1`) with `ansible.builtin.shell` + `sshpass -e` piping IOS config commands directly over SSH via the bastion ProxyCommand (fixes issue #9)
- `inventories/rhdp-sample/group_vars/all.yml` — add `network_host`, `network_username`, `network_password` env var lookups (required for Phase 2 reset)
- `.claude/commands/aiops-reset.md` — extract `NETWORK_HOST`, `NETWORK_USERNAME`, `NETWORK_PASSWORD` from `docs/dev-environment.md` before running reset playbooks

### Claude Code Skills
- `.claude/commands/aiops-preflight.md` — `/aiops-preflight` slash command: reads credentials from `docs/dev-environment.md` and runs `preflight.yml`
- `.claude/commands/aiops-setup.md` — `/aiops-setup` slash command: runs preflight + all three phase setup playbooks in sequence, stops on first failure
- `.claude/commands/aiops-reset.md` — `/aiops-reset` slash command: runs all three reset playbooks in sequence, stops on first failure

### Added
- Initial repo scaffold: README, LICENSE, CODE_OF_CONDUCT, CONTRIBUTING, CHANGELOG
- ansible.cfg, .gitignore, collections/requirements.yml
- Inventory sample (inventories/rhdp-sample/)
- Playbook stubs for preflight, setup, and reset across all three phases
- Project plan document (docs/aiops-workshop-prep-plan.docx)

### Phase 1 — Apache AIOps ✅
- `setup_phase1_apache.yml` — complete and tested against live RHDP
  - Creates "AI Insights and Lightspeed prompt generation" workflow (4 nodes)
  - Creates "Remediation Workflow" (4 nodes)
  - Runs "✅ Restore Apache" to return environment to known-good state

### Phase 2 — Network AIOps ✅
- `setup_phase2_network.yml` — complete and tested against live RHDP
  - Three-step Splunk web auth flow (GET login → POST login → POST Manager endpoint)
    required because OCP only exposes Splunk web port; port 8089 not externally reachable
  - Splunk TCP input port 5514 / `cisco:ios`
  - Launches "Network Router Setup" AAP job template
  - Creates Splunk "ospf-neighbor" real-time alert → EDA webhook
  - Verifies EDA "OSPF Neighbor" rulebook activation is Running
- `reset_phase2_network.yml` — written, not yet tested
  - Restores tunnel0 on cisco-rtr1 via bastion ProxyCommand
- `collections/requirements.yml` — added `ansible.netcommon` and `cisco.ios`
- `inventories/rhdp-sample/group_vars/all.yml` — added `eda_webhook_url`, bastion SSH vars
- `playbooks/preflight.yml` — added "Network Router Setup" job template check

### Phase 3 — Windows AIOps ✅
- `setup_phase3_windows.yml` — complete and tested against live RHDP
  - Verifies 4 Windows job templates exist: "Simulate AD Account Creation",
    "Simulate Windows Firewall Toggle", "Windows: Create Mattermost Ticket",
    "Windows AI: Analyze and Ticket"
  - Verifies "Windows Events" EDA rulebook activation is Running
- `reset_phase3_windows.yml` — complete and tested against live RHDP
  - Re-verifies "Windows Events" EDA activation is Running
  - No physical state to restore (simulate templates leave no lasting changes)
- `playbooks/preflight.yml` — added 4 Windows job template checks

### Security Fixes
- `setup_phase2_network.yml` — added `no_log: true` to both Splunk auth tasks to prevent session tokens appearing in logs (closes #2)
- `reset_phase2_network.yml` — replaced `sshpass -p` with `sshpass -f <tempfile>` so bastion password is not visible in process list; temp file cleaned up in `always:` block (closes #3)
- `reset_phase2_network.yml` — replaced `| default('ansible123!')` with `| mandatory` on `NETWORK_PASSWORD` to fail fast instead of silently using a hardcoded fallback (closes #4)

### Open Issues
- [#1](https://github.com/ericcames/aiops-workshop-prep/issues/1) — Replace Splunk web auth CSRF workaround with SSH tunnel to port 8089 via bastion
