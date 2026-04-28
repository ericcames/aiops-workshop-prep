# Troubleshooting Guide

Common failure modes and how to fix them. All playbooks are idempotent — re-running after fixing the
root cause is always safe.

---

## 1. Wrong RHDP catalog item ordered

**Symptom:** Preflight fails immediately — AAP is unreachable, or job templates are missing.

**Cause:** A different RHDP workshop was provisioned.

**Fix:** Order the correct catalog item:

> **Introduction to AI-Driven Ansible Automation: Self-healing, Observability-Driven AIOps**
> (provided by RHDP)

---

## 2. AAP not reachable

**Symptom:**
```
TASK [Verify AAP is reachable] *****
fatal: [localhost]: FAILED! => {"msg": "...Connection refused..."}
```

**Causes and fixes:**

| Cause | Fix |
|-------|-----|
| Wrong `CONTROLLER_HOST` | Verify the URL from your RHDP instance page — no trailing slash |
| RHDP instance expired | Extend or re-provision from the RHDP catalog |
| Instance still provisioning | Wait a few minutes and retry |
| Wrong `CONTROLLER_PASSWORD` | Re-check the password from the RHDP instance details page |

---

## 3. Job template not found (name mismatch)

**Symptom:**
```
TASK [Check required job templates exist] *****
fatal: [localhost]: FAILED! => {"msg": "...Could not find job_template..."}
```

**Cause:** The upstream lab provisioned a template with a slightly different name (emoji, spacing,
or capitalisation difference).

**Fix:**
1. Log into the AAP UI at `CONTROLLER_HOST`
2. Go to **Templates** and note the exact name
3. Update the template name in the relevant playbook task

**Known template names** (as of Summit 2025 lab):
- `❌ Break Apache`
- `✅ Restore Apache`
- `⚙️ Apache Service Status Check`
- `🤖 RHEL AI: Analyze Incident`
- `📣 Notify via Mattermost`
- `⚙️ Build Ansible Lightspeed Job Template`
- `🧾 Commit Fix to Gitea`
- `⚙️ Build HTTPD Remediation Template`
- `Network Router Setup`
- `Simulate AD Account Creation`
- `Simulate Windows Firewall Toggle`
- `Windows: Create Mattermost Ticket`
- `Windows AI: Analyze and Ticket`

---

## 4. Splunk authentication fails (Phase 2 setup)

**Symptom:**
```
TASK [Splunk auth — post credentials to establish session] *****
fatal: [localhost]: FAILED!
```
or the subsequent Splunk tasks return `403` or `401`.

**Cause:** Wrong `SPLUNK_PASSWORD`, expired session, or CSRF token mismatch.

**Note:** OCP only exposes the Splunk **web port (443)** — the management API on port 8089 is not
externally reachable. The playbook uses a 3-step web auth flow (GET login → POST credentials →
POST to Manager endpoint with `splunk_form_key`). Basic auth and Bearer tokens do not work via
the web port.

**Fix:**
1. Verify `SPLUNK_HOST` and `SPLUNK_PASSWORD` match the RHDP instance details
2. Re-run `setup_phase2_network.yml` — the 3-step auth flow fetches a fresh CSRF token each time

---

## 5. EDA rulebook activation not in Running state

**Symptom:**
```
TASK [Verify EDA OSPF Neighbor rulebook activation is Running] *****
fatal: [localhost]: FAILED! => {"msg": "...status is 'failed' not 'running'..."}
```
or similar for `Windows Events`.

**Cause:** The EDA activation crashed, was manually stopped, or the upstream lab setup didn't
complete.

**Fix:**
1. Log into the AAP UI → **EDA** → **Rulebook Activations**
2. Find the activation (`OSPF Neighbor` or `Windows Events`)
3. If status is `Failed` or `Stopped`: click **Restart**
4. Wait ~30 seconds for it to reach `Running` state
5. Re-run the setup or reset playbook

---

## 6. Bastion SSH connection fails (Phase 2 reset)

**Symptom:**
```
TASK [Restore tunnel0 on cisco-rtr1 (no shut)] *****
fatal: [localhost]: FAILED! => {"msg": "...Connection refused..."}
```
or timeout on the reset task.

**Causes and fixes:**

| Cause | Fix |
|-------|-----|
| Wrong `BASTION_HOST` / `BASTION_PORT` | Re-check from RHDP instance details — format is `ssh.<cluster>.rhdp.net` / port number |
| Wrong `BASTION_PASSWORD` | Verify from RHDP instance details |
| RHDP instance expired | Extend or re-provision |

**Test the bastion manually:**
```bash
sshpass -p '<bastion-password>' ssh -p <port> -o StrictHostKeyChecking=no \
  lab-user@<bastion-host> "echo ok"
```

**Test the full chain to cisco-rtr1:**
```bash
sshpass -p '<cisco-password>' ssh \
  -o "ProxyCommand=sshpass -p '<bastion-password>' ssh -p <port> -o StrictHostKeyChecking=no lab-user@<bastion-host> -W %h:%p" \
  -o StrictHostKeyChecking=no \
  admin@cisco-rtr1 "show ip interface brief tunnel0"
```

---

## 7. Collections not installed

**Symptom:**
```
ERROR! the role 'ansible.platform' was not found
```
or
```
ModuleNotFoundError: No module named 'ansible_collections.ansible.platform'
```

**Fix:**
```bash
ANSIBLE_CONFIG=~/.ansible/ansible.cfg \
  ansible-galaxy collection install -r collections/requirements.yml -p ./collections
```

Requires a valid Automation Hub token in `~/.ansible/ansible.cfg` under `[galaxy_server.rh_certified]`.

---

## 8. `docs/dev-environment.md` not found

**Symptom:** The `/aiops-preflight`, `/aiops-setup`, or `/aiops-reset` Claude Code skill stops
immediately with:
```
❌ docs/dev-environment.md not found.
```

**Fix:**
```bash
cp docs/dev-environment.md.example docs/dev-environment.md
# then edit docs/dev-environment.md and fill in your RHDP credentials
```

---

## 9. Splunk alert not firing during Network AIOps demo

**Symptom:** You shut tunnel0 on cisco-rtr1 but the OSPF Neighbor alert never fires in Splunk
and EDA doesn't trigger.

**Causes and fixes:**

| Cause | Fix |
|-------|-----|
| Splunk TCP input not created | Re-run `setup_phase2_network.yml` |
| Cisco IOS not sending syslog to Splunk | Run "Network Router Setup" job template in AAP to configure syslog forwarding |
| ospf-neighbor alert not created or disabled | Log into Splunk → **Alerts** and verify `ospf-neighbor` alert exists and is enabled |
| EDA webhook URL wrong in Splunk alert | Re-run `setup_phase2_network.yml` to recreate the alert with the correct webhook URL |

---

## 10. All three phases pass setup but demo doesn't work end-to-end

After setup completes, verify the environment with preflight:
```bash
ansible-playbook -i inventories/rhdp-sample/ playbooks/preflight.yml
```

Then run a quick smoke test for each phase:

| Phase | Smoke test |
|-------|-----------|
| Apache | Launch "❌ Break Apache" in AAP — watch the EDA rulebook trigger the AI Insights workflow |
| Network | SSH to cisco-rtr1 via bastion, run `conf t; int tunnel0; shutdown; end` — watch Splunk alert fire → EDA webhook → Network-AIOps-Workflow |
| Windows | Launch "Simulate AD Account Creation" in AAP — watch the Windows Events EDA activation trigger "Windows: Create Mattermost Ticket" |
