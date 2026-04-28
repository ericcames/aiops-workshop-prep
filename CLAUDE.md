# aiops-workshop-prep — Claude Guidelines

## What This Repo Does

Automates the setup of the [ansible-tmm/aiops-summitlab](https://github.com/ansible-tmm/aiops-summitlab)
AIOps workshop so each demo section is ready to run without manual steps.

**RHDP Catalog Item:** "Introduction to AI-Driven Ansible Automation: Self-healing, Observability-Driven AIOps" (provided by RHDP)

The upstream repo must be deployed to RHDP before running anything here.

## Open Issues

Track work items at https://github.com/ericcames/aiops-workshop-prep/issues

| # | Title | Priority |
|---|-------|----------|
| #13 | Automate Splunk API token creation for Part 2 setup | Medium |
| #5 | validate_certs: false — should validate in prod | Low |
| #6 | SSH host key checking disabled — should use known_hosts | Low |

Issues #2, #3, #4 (credential exposure) closed in commit f877947.
Issues #8 (Claude Code skills), #9 (Phase 2 reset Paramiko bug) closed in PR #10.

## Current State

### Phase 1 — Apache AIOps ✅ COMPLETE AND TESTED
`playbooks/setup_phase1_apache.yml` — working against live RHDP, all 3 tasks pass:
- Creates "AI Insights and Lightspeed prompt generation" workflow (4 nodes)
- Creates "Remediation Workflow" (4 nodes)
- Runs "✅ Restore Apache" to return environment to known-good state

`playbooks/reset_phase1_apache.yml` — ✅ tested: runs "✅ Restore Apache"

### Phase 2 — Network AIOps (Splunk + Cisco + EDA) ✅ COMPLETE AND TESTED
`playbooks/setup_phase2_network.yml` — working against live RHDP, all 6 tasks pass:
- Authenticates to Splunk via 3-step web auth (GET login → POST login → POST Manager endpoint)
- Creates Splunk TCP input port 5514 / `cisco:ios`
- Launches "Network Router Setup" job template
- Creates Splunk "ospf-neighbor" real-time alert → EDA webhook
- Verifies EDA "OSPF Neighbor" rulebook activation is Running

`playbooks/reset_phase2_network.yml` — ✅ tested: restores tunnel0 on cisco-rtr1 via `ansible.builtin.shell` + `sshpass -e` piping IOS config commands over SSH through the bastion ProxyCommand.

**Splunk note:** OCP only exposes Splunk web port (443); port 8089 REST API is not reachable externally.
The web port has CSRF protection. Solution is the Manager controller endpoint at `/en-US/manager/`
which accepts `splunk_form_key` in the POST body and strips it before calling splunkd.
Basic auth and Bearer tokens do not work via the web port.

**Phase 2 reset note:** `ansible.netcommon.cli_config` with Paramiko cannot resolve the internal
`cisco-rtr1` hostname and does not apply the SSH ProxyCommand. Use `ansible.builtin.shell` with
`sshpass -e` instead (already implemented).

### Phase 3 — Windows AIOps ✅ COMPLETE AND TESTED
`playbooks/setup_phase3_windows.yml` — working against live RHDP, all 2 tasks pass:
- Verifies all 4 Windows job templates exist: "Simulate AD Account Creation", "Simulate Windows Firewall Toggle", "Windows: Create Mattermost Ticket", "Windows AI: Analyze and Ticket"
- Verifies "Windows Events" EDA rulebook activation is Running

`playbooks/reset_phase3_windows.yml` — working against live RHDP:
- Re-verifies "Windows Events" EDA activation is Running (nothing physically breaks during simulate templates — no state to restore)

**Windows AIOps note:** Infrastructure (Windows Server, Winlogbeat, Kafka) is pre-provisioned by RHDP. Setup is verification-only. Demo entry points are "Simulate AD Account Creation" (triggers Mattermost ticket) and "Simulate Windows Firewall Toggle" (triggers AI analysis + enriched ticket).

## Running the Playbooks

Credentials live in `docs/dev-environment.md` (gitignored — never commit).
Set environment variables before running:

```bash
export CONTROLLER_HOST=<aap-url>
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD=<password>
export SPLUNK_HOST=<splunk-url>
export SPLUNK_USERNAME=admin
export SPLUNK_PASSWORD=<password>
export EDA_WEBHOOK_URL=<eda-webhook-url>
export BASTION_HOST=<bastion-host>
export BASTION_PORT=<bastion-port>
export BASTION_USER=lab-user
export BASTION_PASSWORD=<password>

# Validate upstream lab is deployed
ansible-playbook -i inventories/rhdp-sample/ playbooks/preflight.yml

# Setup
ansible-playbook -i inventories/rhdp-sample/ playbooks/setup_phase1_apache.yml
ansible-playbook -i inventories/rhdp-sample/ playbooks/setup_phase2_network.yml
ansible-playbook -i inventories/rhdp-sample/ playbooks/setup_phase3_windows.yml

# Reset between sessions
ansible-playbook -i inventories/rhdp-sample/ playbooks/reset_phase1_apache.yml
ansible-playbook -i inventories/rhdp-sample/ playbooks/reset_phase2_network.yml
ansible-playbook -i inventories/rhdp-sample/ playbooks/reset_phase3_windows.yml
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

| Showroom Section | Demo trigger | EDA/workflow |
|-----------------|-------------|--------------|
| Part 1: AIOps with Apache Remediation | Run "❌ Break Apache" | Web App rulebook → AI Insights workflow |
| Part 2: AIOps with Network Automation | SSH cisco-rtr1, shut tunnel0 | ospf-neighbor Splunk alert → OSPF Neighbor rulebook → Network-AIOps-Workflow |
| Part 3: AIOps with Windows Automation | Launch "Simulate AD Account Creation" or "Simulate Windows Firewall Toggle" | Windows Events rulebook → "Windows: Create Mattermost Ticket" or "Windows AI: Analyze and Ticket" |

## Claude Code Skills

Three slash commands in `.claude/commands/` automate the common workflows. They read credentials
from `docs/dev-environment.md` automatically — no manual `export` needed:

| Command | What it does |
|---------|-------------|
| `/aiops-preflight` | Validates AAP + Splunk + EDA are ready before a demo |
| `/aiops-setup` | Runs preflight + all three phase setup playbooks in sequence |
| `/aiops-reset` | Resets all three phases to known-good state between sessions |

## Conventions

- Use `ansible.platform` modules where available; `ansible.controller` for workflow modules
- All playbooks must be idempotent
- Update `CHANGELOG.md` before every commit
- One fix per branch and PR
- Any playbook that creates a token must delete it in an `always:` block
