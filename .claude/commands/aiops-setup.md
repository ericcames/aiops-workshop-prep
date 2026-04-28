---
name: aiops-setup
description: "Full setup sequence for a fresh RHDP AIOps workshop environment. Reads credentials from docs/dev-environment.md, runs preflight, then sets up all three phases (Apache, Network, Windows) in sequence. Stops on first failure. TRIGGER when: user has a fresh RHDP deployment and wants to run the full workshop setup."
---

# AIOps Setup

Full setup sequence for a fresh RHDP AIOps workshop environment. Runs preflight then all three phase setup playbooks in order.

## Step 1 — Check Credentials File

Verify `docs/dev-environment.md` exists:

```bash
test -f docs/dev-environment.md && echo "✅ docs/dev-environment.md found" || echo "❌ docs/dev-environment.md not found"
```

If the file does not exist, stop and tell the user:

> ❌ docs/dev-environment.md not found. Copy docs/dev-environment.md.example and fill in your RHDP instance credentials before running setup.

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
