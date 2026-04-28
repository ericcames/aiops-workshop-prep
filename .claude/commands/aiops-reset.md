---
name: aiops-reset
description: "Resets all three AIOps workshop phases to a known-good state between customer sessions. Reads credentials from docs/dev-environment.md and runs all three reset playbooks. TRIGGER when: user wants to return the workshop to a known-good state before the next demo session."
---

# AIOps Reset

Resets all three workshop phases to a known-good state. Run between customer sessions.

## Step 1 — Check Credentials File

Verify `docs/dev-environment.md` exists:

```bash
test -f docs/dev-environment.md && echo "✅ docs/dev-environment.md found" || echo "❌ docs/dev-environment.md not found"
```

If the file does not exist, stop and tell the user:

> ❌ docs/dev-environment.md not found. Copy docs/dev-environment.md.example and fill in your RHDP instance credentials before running reset.

## Step 2 — Run All Reset Playbooks

Read `docs/dev-environment.md` and extract credentials using this table, then run all three reset playbooks — stopping immediately if any fails:

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
| `NETWORK_HOST` | `## SSH (cisco-rtr1 — via bastion)` | `Internal IP` |
| `NETWORK_USERNAME` | `## SSH (cisco-rtr1 — via bastion)` | `Username` |
| `NETWORK_PASSWORD` | `## SSH (cisco-rtr1 — via bastion)` | `Password` |

> **Note — Phase 2 reset:** `reset_phase2_network.yml` reconnects tunnel0 on cisco-rtr1 via the SSH bastion. It requires `BASTION_HOST`, `BASTION_PORT`, `BASTION_USER`, and `BASTION_PASSWORD` to be set correctly.

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
    'NETWORK_HOST':        get('SSH (cisco-rtr1 — via bastion)', 'Internal IP'),
    'NETWORK_USERNAME':    get('SSH (cisco-rtr1 — via bastion)', 'Username'),
    'NETWORK_PASSWORD':    get('SSH (cisco-rtr1 — via bastion)', 'Password'),
}

steps = [
    ('Phase 1 — Apache AIOps', 'playbooks/reset_phase1_apache.yml'),
    ('Phase 2 — Network AIOps','playbooks/reset_phase2_network.yml'),
    ('Phase 3 — Windows AIOps','playbooks/reset_phase3_windows.yml'),
]

results = []
for label, playbook in steps:
    print(f'\n{"="*60}')
    print(f'Resetting {label}...')
    print('='*60)
    r = subprocess.run(
        ['ansible-playbook', '-i', 'inventories/rhdp-sample/', playbook],
        env=env
    )
    if r.returncode != 0:
        print(f'\n❌ {label} reset failed — stopping.')
        for done_label, _ in results:
            print(f'  ✅ {done_label}')
        print(f'  ❌ {label} — FAILED')
        sys.exit(r.returncode)
    results.append((label, playbook))
    print(f'✅ {label} reset complete')

print('\n' + '='*60)
print('Reset complete. All phases are in known-good state.\n')
for label, _ in results:
    print(f'  ✅ {label}')
print('\nReady for the next demo session.')
EOF
```

## Step 3 — On Failure

Show which phase failed and display the Ansible error output. Suggest re-running the single failed reset playbook directly after fixing the issue:

```bash
ansible-playbook -i inventories/rhdp-sample/ playbooks/<failed-reset-playbook>.yml
```
