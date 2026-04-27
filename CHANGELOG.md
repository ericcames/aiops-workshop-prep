# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

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
