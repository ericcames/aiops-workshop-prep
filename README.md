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
- Collections installed (requires Automation Hub token in `~/.ansible/ansible.cfg`):
  ```bash
  ANSIBLE_CONFIG=~/.ansible/ansible.cfg \
    ansible-galaxy collection install -r collections/requirements.yml -p ./collections
  ```
- Credentials populated in `docs/dev-environment.md` (gitignored — never commit)

## Environment Variables

Set these before running any playbook:

```bash
# AAP
export CONTROLLER_HOST=<aap-url>
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD=<password>

# Splunk (Phase 2)
export SPLUNK_HOST=<splunk-url>
export SPLUNK_USERNAME=admin
export SPLUNK_PASSWORD=<password>

# EDA webhook (Phase 2)
export EDA_WEBHOOK_URL=<eda-webhook-url>

# Bastion SSH — required to reach cisco-rtr1 for Phase 2 reset
export BASTION_HOST=<bastion-host>
export BASTION_PORT=<bastion-port>
export BASTION_USER=lab-user
export BASTION_PASSWORD=<password>
```

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

Copy the sample inventory for each new RHDP environment and set the environment variables above:

```bash
cp -r inventories/rhdp-sample/ inventories/rhdp-<customer>/
```

## Phases

| Phase | Section | Status | What it automates |
|---|---|---|---|
| Phase 1 | Apache AIOps | ✅ Tested | Builds AI Insights and Remediation workflows |
| Phase 2 | Network AIOps | ✅ Tested | Splunk TCP input, Network Router Setup job, Splunk alert → EDA webhook |
| Phase 3 | Windows AIOps | ✅ Tested | Verifies Windows job templates and EDA activation |

## Claude Code Skills

If you're using [Claude Code](https://claude.ai/code), three slash commands are available that read credentials automatically from `docs/dev-environment.md`:

| Command | What it does |
|---------|-------------|
| `/aiops-preflight` | Validates the environment is ready before a demo |
| `/aiops-setup` | Runs preflight + all three phase setup playbooks in sequence |
| `/aiops-reset` | Resets all three phases to known-good state |

No manual `export` of environment variables needed — each command reads `docs/dev-environment.md` directly.

## License

MIT — see [LICENSE](LICENSE)
