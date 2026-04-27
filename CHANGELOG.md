# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Added
- Initial repo scaffold: README, LICENSE, CODE_OF_CONDUCT, CONTRIBUTING, CHANGELOG
- ansible.cfg, .gitignore, collections/requirements.yml
- Inventory sample (inventories/rhdp-sample/)
- Playbook stubs for preflight, setup, and reset across all three phases
- Project plan document (docs/aiops-workshop-prep-plan.docx)

### Phase 1 — Apache AIOps
- `setup_phase1_apache.yml` — complete and tested against live RHDP

### Phase 2 — Network AIOps
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
