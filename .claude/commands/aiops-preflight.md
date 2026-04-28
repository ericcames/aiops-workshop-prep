---
name: aiops-preflight
description: "Quick validation before an AIOps workshop demo. Reads credentials from docs/dev-environment.md, runs preflight.yml, and reports whether all required AAP templates and EDA activations are present. TRIGGER when: user wants to verify the workshop environment is ready before a demo."
---

# AIOps Preflight

Validates that the RHDP AIOps workshop environment is ready before a demo.

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
