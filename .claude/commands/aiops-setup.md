---
name: aiops-setup
description: "Full setup sequence for a fresh RHDP AIOps workshop environment. Reads credentials from docs/dev-environment.md, runs preflight, then sets up all three phases (Apache, Network, Windows) in sequence. Stops on first failure. TRIGGER when: user has a fresh RHDP deployment and wants to run the full workshop setup."
---

# AIOps Setup

Full setup sequence for a fresh RHDP AIOps workshop environment. Runs preflight then all three phase setup playbooks in order.

## Step 1 — Check Credentials File

```bash
test -f docs/dev-environment.md && echo "found" || echo "missing"
```

If the file exists, proceed to Step 2.

If the file is missing, tell the user you'll help create it and ask them to provide all of the following values in one reply (they can find these on their RHDP instance details page):

- AAP URL (e.g. `https://controller-xxxx.apps.cluster.rhdp.net`)
- AAP password (username is always `admin`)
- Splunk URL (e.g. `https://splunk-xxxx.apps.cluster.rhdp.net`)
- Splunk password (username is always `admin`)
- Splunk API token (Splunk UI → Settings → Tokens → New Token — needed for Part 2 demo)
- EDA Webhook URL (e.g. `https://eda-webhook-xxxx.apps.cluster.rhdp.net`)
- SSH Bastion host (e.g. `ssh.cluster.rhdp.net`)
- SSH Bastion port (a 5-digit number)
- SSH Bastion password (username is always `lab-user`)
- Cisco router internal IP (e.g. `10.x.x.x`)
- Cisco router password (username is always `admin`)

Once the user provides the values, write `docs/dev-environment.md` with this exact format, substituting the actual values:

```
# Dev Environment — Current RHDP Instance

This file is gitignored. Never commit it.

## AAP

- **URL:** <aap-url>
- **Username:** admin
- **Password:** <aap-password>

## Splunk

- **URL:** <splunk-url>
- **Username:** admin
- **Password:** <splunk-password>
- **API Token:** <splunk-api-token>
  (not used by setup playbooks — for Part 2 demo reference; see issue #1 to automate)

## EDA Webhook

- **URL:** <eda-webhook-url>

## SSH Bastion

- **Host:** <bastion-host>
- **Port:** <bastion-port>
- **Username:** lab-user
- **Password:** <bastion-password>

## SSH (cisco-rtr1 — via bastion)

- **Internal IP:** <cisco-internal-ip>
- **Username:** admin
- **Password:** <cisco-password>
```

Confirm the file was written, then proceed to Step 2.

## Step 2 — Run Full Setup Sequence

Read `docs/dev-environment.md` and extract credentials using this table, then run all four playbooks in sequence — stopping immediately if any fails:

| Variable | Section | Field |
|----------|---------|-------|
| `CONTROLLER_HOST` | `## AAP` | `URL` |
| `CONTROLLER_USERNAME` | `## AAP` | `Username` |
| `CONTROLLER_PASSWORD` | `## AAP` | `Password` |
| `SPLUNK_HOST` | `## Splunk` | `URL` |
| `SPLUNK_USERNAME` | `## Splunk` | `Username` |
| `SPLUNK_PASSWORD` | `## Splunk` | `Password` |
| `EDA_WEBHOOK_URL` | `## EDA Webhook` | `URL` |
| `BASTION_HOST` | `## SSH Bastion` | `Host` |
| `BASTION_PORT` | `## SSH Bastion` | `Port` |
| `BASTION_USER` | `## SSH Bastion` | `Username` |
| `BASTION_PASSWORD` | `## SSH Bastion` | `Password` |

```bash
python3 << 'EOF'
import re, subprocess, os, sys

content = open('docs/dev-environment.md').read()

def get(section, field):
    m = re.search(rf'^## {re.escape(section)}$', content, re.MULTILINE)
    if not m:
        return ''
    rest = content[m.end():]
    n = re.search(r'^## ', rest, re.MULTILINE)
    sec = rest[:n.start() if n else len(rest)]
    fm = re.search(rf'\*\*{re.escape(field)}:\*\*\s*(.+)', sec)
    return fm.group(1).strip() if fm else ''

env = {**os.environ,
    'CONTROLLER_HOST':     get('AAP', 'URL'),
    'CONTROLLER_USERNAME': get('AAP', 'Username'),
    'CONTROLLER_PASSWORD': get('AAP', 'Password'),
    'SPLUNK_HOST':         get('Splunk', 'URL'),
    'SPLUNK_USERNAME':     get('Splunk', 'Username'),
    'SPLUNK_PASSWORD':     get('Splunk', 'Password'),
    'EDA_WEBHOOK_URL':     get('EDA Webhook', 'URL'),
    'BASTION_HOST':        get('SSH Bastion', 'Host'),
    'BASTION_PORT':        get('SSH Bastion', 'Port'),
    'BASTION_USER':        get('SSH Bastion', 'Username'),
    'BASTION_PASSWORD':    get('SSH Bastion', 'Password'),
}

steps = [
    ('Preflight',              'playbooks/preflight.yml'),
    ('Phase 1 — Apache AIOps', 'playbooks/setup_phase1_apache.yml'),
    ('Phase 2 — Network AIOps','playbooks/setup_phase2_network.yml'),
    ('Phase 3 — Windows AIOps','playbooks/setup_phase3_windows.yml'),
]

results = []
for label, playbook in steps:
    print(f'\n{"="*60}')
    print(f'Running {label}...')
    print('='*60)
    r = subprocess.run(
        ['ansible-playbook', '-i', 'inventories/rhdp-sample/', playbook],
        env=env
    )
    if r.returncode != 0:
        print(f'\n❌ {label} failed — stopping.')
        for done_label, _ in results:
            print(f'  ✅ {done_label}')
        print(f'  ❌ {label} — FAILED')
        sys.exit(r.returncode)
    results.append((label, playbook))
    print(f'✅ {label} complete')

print('\n' + '='*60)
print('Setup complete.\n')
for label, _ in results:
    print(f'  ✅ {label}')
print("""
Demo entry points:
  Apache:  Run "❌ Break Apache" in AAP
  Network: SSH cisco-rtr1, shut tunnel0
  Windows: Launch "Simulate AD Account Creation" or "Simulate Windows Firewall Toggle"
""")
EOF
```

## Step 3 — On Failure

Show which phase failed and display the Ansible error output. Do not suggest re-running the full sequence — suggest re-running the single failed playbook directly after fixing the issue:

```bash
ansible-playbook -i inventories/rhdp-sample/ playbooks/<failed-playbook>.yml
```
