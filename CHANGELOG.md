# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Added
- `eda_webhook_url` env var lookup to inventories/rhdp-sample/group_vars/all.yml for Phase 2 Network AIOps
- `ansible.netcommon` and `cisco.ios` collections to requirements.yml for Phase 2 network reset playbook
- "Network-Router-Setup" job template check to preflight.yml
- Initial repo scaffold: README, LICENSE, CODE_OF_CONDUCT, CONTRIBUTING, CHANGELOG
- ansible.cfg, .gitignore, collections/requirements.yml
- Inventory sample (inventories/rhdp-sample/)
- Playbook stubs for preflight, setup, and reset across all three phases
- Project plan document (docs/aiops-workshop-prep-plan.docx)
