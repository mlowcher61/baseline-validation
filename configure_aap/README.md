# AAP Config-as-Code

This directory contains everything needed to stand up the AAP objects for the baseline validation demo using a single playbook run.

## What it creates

| Object | Name |
|--------|------|
| Organization | Demo |
| Project | Baseline Validation |
| Inventory | Baseline Demo Inventory |
| Machine Credential | Baseline Demo Machine Credential |
| Job Template + Survey | Validate Baseline |

## Prerequisites

1. Install required collections:
   ```bash
   ansible-galaxy collection install -r collections/requirements.yml
   ```

2. Export AAP controller connection details (no secrets in git):
   ```bash
   export CONTROLLER_HOST=https://your-aap-controller.example.com
   export CONTROLLER_USERNAME=admin
   export CONTROLLER_PASSWORD=yourpassword
   # Or use a token instead:
   # export CONTROLLER_OAUTH_TOKEN=yourtoken
   ```

3. After `configure.yml` runs, open the AAP UI and add the actual SSH username/key to the **Baseline Demo Machine Credential** — the playbook creates the credential object but cannot store secrets.

4. Add your target RHEL hosts to the **Baseline Demo Inventory** in the AAP UI (or wire up a dynamic inventory source).

## Run

```bash
ansible-playbook configure_aap/configure.yml
```

The playbook is idempotent — safe to re-run after any change to the vars files.
