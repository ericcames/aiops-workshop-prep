# Contributing

Contributions are welcome. Please follow these guidelines:

- Open a GitHub Issue before making code changes
- One fix or feature per branch and PR
- All playbooks must be idempotent
- Update CHANGELOG.md in every PR
- Never commit credentials or tokens — use environment variables or `docs/dev-environment.md` (gitignored)
- Use `ansible.platform` modules where available; fall back to `ansible.controller` only when no platform equivalent exists

## Getting Started

1. Fork the repo and clone locally
2. Install collections: `ansible-galaxy collection install -r collections/requirements.yml`
3. Copy the sample inventory: `cp -r inventories/rhdp-sample/ inventories/rhdp-<your-env>/`
4. Populate `docs/dev-environment.md` with your environment credentials
5. Run `ansible-playbook -i inventories/rhdp-<your-env>/ playbooks/preflight.yml` to validate your environment
