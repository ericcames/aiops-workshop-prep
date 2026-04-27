# aiops-workshop-prep — Claude Guidelines

## What This Repo Does

Automates the setup of the [ansible-tmm/aiops-summitlab](https://github.com/ansible-tmm/aiops-summitlab)
AIOps workshop so each demo section is ready to run without manual steps.
The upstream repo must be deployed to RHDP before running anything here.

## Current State

### Phase 1 — Apache AIOps ✅ COMPLETE AND TESTED
`playbooks/setup_phase1_apache.yml` — working against live RHDP, all 3 tasks pass:
- Creates "AI Insights and Lightspeed prompt generation" workflow (4 nodes)
- Creates "Remediation Workflow" (4 nodes)
- Runs "✅ Restore Apache" to return environment to known-good state

`playbooks/reset_phase1_apache.yml` — runs "✅ Restore Apache", not yet tested

### Phase 2 — Network AIOps (Splunk + Cisco + EDA) ⬜ STUB ONLY
`playbooks/setup_phase2_network.yml` — logic written but NOT yet tested:
- Configures Splunk TCP input (port 5514, cisco:ios) via REST API
- Launches "Network-Router-Setup" job template
- Creates Splunk "ospf-neighbor" real-time alert → EDA webhook
- Verifies EDA "OSPF Neighbor" rulebook activation is Running

`playbooks/reset_phase2_network.yml` — stub, NOT yet tested

### Phase 3 — Windows AIOps ⬜ PLACEHOLDER
Both setup and reset playbooks are debug message placeholders.
Need to review the Windows module content from the showroom before implementing.

## Running the Playbooks

Credentials live in `docs/dev-environment.md` (gitignored — never commit).
Set environment variables before running:

```bash
export CONTROLLER_HOST=<aap-url>
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD=<password>

# Validate upstream lab is deployed
ansible-playbook -i inventories/rhdp-sample/ playbooks/preflight.yml

# Setup
ansible-playbook -i inventories/rhdp-sample/ playbooks/setup_phase1_apache.yml
ansible-playbook -i inventories/rhdp-sample/ playbooks/setup_phase2_network.yml

# Reset between sessions
ansible-playbook -i inventories/rhdp-sample/ playbooks/reset_phase1_apache.yml
```

## Collections

Collections are installed locally but gitignored. Install with:

```bash
ANSIBLE_CONFIG=~/.ansible/ansible.cfg \
  ansible-galaxy collection install -r collections/requirements.yml -p ./collections
```

Requires a valid Automation Hub token in `~/.ansible/ansible.cfg`.

## Key Decisions Made

- **Separate repo** from `aap.as.code` (general bootstrap) and `ansible-tmm/aiops-summitlab` (upstream lab content)
- **ansible.controller** handles workflow modules — `ansible.platform` does not have them
- **AAP 4.x ping endpoint** is `/api/controller/v2/ping/` not `/api/v2/ping/`
- **Credentials** stored in `docs/dev-environment.md` (gitignored) — never commit

## Upstream Lab Structure

The upstream `ansible-tmm/aiops-summitlab` repo has:
- `playbooks/` — all job template playbooks
- `setup_playbooks/` — minimal upstream setup (Kafka, Filebeat, one workflow)
- `rulebooks/` — EDA rulebooks including `ospf.yml`

## Lab Sections and Demo Entry Points

| Section | Demo trigger | EDA/workflow |
|---------|-------------|--------------|
| Apache AIOps | Run "❌ Break Apache" | Web App rulebook → AI Insights workflow |
| Network AIOps | SSH cisco-rtr1, shut tunnel0 | ospf-neighbor Splunk alert → OSPF Neighbor rulebook → Network-AIOps-Workflow |
| Windows AIOps | TBD | TBD |

## Conventions

- Use `ansible.platform` modules where available; `ansible.controller` for workflow modules
- All playbooks must be idempotent
- Update `CHANGELOG.md` before every commit
- One fix per branch and PR
- Any playbook that creates a token must delete it in an `always:` block
