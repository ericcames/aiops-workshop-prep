# aiops-workshop-prep

Automates the setup of the [ansible-tmm/aiops-summitlab](https://github.com/ansible-tmm/aiops-summitlab) AIOps workshop so each demo section is ready to run without manual steps.

## Upstream Dependency

This repo targets a running instance of the upstream lab. The upstream repo must be deployed to your RHDP environment before running anything here.

| | |
|---|---|
| **Upstream repo** | https://github.com/ansible-tmm/aiops-summitlab |
| **Description** | Summit 2025 AIOps Lab — source of all job templates, rulebooks, and playbooks |

## Prerequisites

- A running RHDP AIOps workshop environment
- Ansible installed locally
- Collections installed: `ansible-galaxy collection install -r collections/requirements.yml`
- Credentials populated in `docs/dev-environment.md` (see `docs/dev-environment.md.example` — never commit this file)

## Usage

Each phase corresponds to a section of the workshop. Run setup before the demo, reset between customer sessions.

```bash
# Validate the upstream lab is deployed correctly
ansible-playbook -i inventories/rhdp-<customer>/ playbooks/preflight.yml

# Set up each section
ansible-playbook -i inventories/rhdp-<customer>/ playbooks/setup_phase1_apache.yml
ansible-playbook -i inventories/rhdp-<customer>/ playbooks/setup_phase2_network.yml
ansible-playbook -i inventories/rhdp-<customer>/ playbooks/setup_phase3_windows.yml

# Reset between sessions
ansible-playbook -i inventories/rhdp-<customer>/ playbooks/reset_phase1_apache.yml
ansible-playbook -i inventories/rhdp-<customer>/ playbooks/reset_phase2_network.yml
ansible-playbook -i inventories/rhdp-<customer>/ playbooks/reset_phase3_windows.yml
```

## Inventory Setup

Copy the sample inventory for each new RHDP environment:

```bash
cp -r inventories/rhdp-sample/ inventories/rhdp-<customer>/
export CONTROLLER_HOST=<aap-url>
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD=<password>
```

## Phases

| Phase | Section | What it automates |
|---|---|---|
| Phase 1 | Apache AIOps | Builds AI Insights and Remediation workflows |
| Phase 2 | Network AIOps | Configures Splunk, Cisco routers, EDA |
| Phase 3 | Windows AIOps | Sets up Windows EDA, Mattermost tickets |

## License

MIT — see [LICENSE](LICENSE)
