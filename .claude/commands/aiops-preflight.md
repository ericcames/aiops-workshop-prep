---
name: aiops-preflight
description: "Quick validation before an AIOps workshop demo. Reads credentials from docs/dev-environment.md, runs preflight.yml, and reports whether all required AAP templates and EDA activations are present. TRIGGER when: user wants to verify the workshop environment is ready before a demo."
---

# AIOps Preflight

Validates that the RHDP AIOps workshop environment is ready before a demo.

## Step 1 — Check Credentials File

Verify `docs/dev-environment.md` exists:

```bash
test -f docs/dev-environment.md && echo "✅ docs/dev-environment.md found" || echo "❌ docs/dev-environment.md not found"
```

If the file does not exist, stop and tell the user:

> ❌ docs/dev-environment.md not found. Copy docs/dev-environment.md.example and fill in your RHDP instance credentials before running preflight.

## Step 2 — Run Preflight

Read `docs/dev-environment.md` and extract credentials using this table, then run `preflight.yml` with those values set as environment variables:

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

result = subprocess.run(
    ['ansible-playbook', '-i', 'inventories/rhdp-sample/', 'playbooks/preflight.yml'],
    env=env
)
sys.exit(result.returncode)
EOF
```

## Step 3 — Report Results

On success:

```
✅ Preflight passed — workshop environment is ready.

  AAP:         <CONTROLLER_HOST>
  Splunk:      <SPLUNK_HOST>
  EDA Webhook: <EDA_WEBHOOK_URL>
```

On failure, show the specific task that failed from Ansible output and the most likely fix.
