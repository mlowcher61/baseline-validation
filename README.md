# Baseline Validation

An Ansible Automation Platform (AAP) solution that validates whether a RHEL host meets a defined technical baseline and produces a clear **PASS / NON-COMPLIANT** report in the job output.

## What it does

When you run the **Validate Baseline** job template against a RHEL host, the playbook:

1. Gathers live facts from the target host, including SSH service state and effective `sshd` configuration.
2. Compares the baseline parameters — OS version, CPU cores, memory, root filesystem, and three SSH security controls — against configurable expectations.
3. Prints a report in the job output — every check, pass or fail, with the actual value found.
4. Fails the job (red in AAP) if any check fails, so you get an immediate visual signal.

Example output for a non-compliant host:

```
BASELINE VALIDATION — web01.example.com
=========================================
[ PASS ] CPU cores                       : 4    (min 2 cores)
[ FAIL ] OS version                      : 8.6  (min 9.0)   <-- does not meet validated baseline
[ PASS ] Memory                          : 8.0  (min 4 GB)
[ PASS ] Root filesystem capacity        : 80   (min 40 GB)
[ PASS ] SSH service                     : yes  (expected yes (enabled+running))
[ PASS ] Password authentication disabled: no   (expected no)
[ FAIL ] Root login disabled             : yes  (expected no)   <-- does not meet validated baseline
-----------------------------------------
RESULT: NON-COMPLIANT  (2 of 7 checks failed)
```

## Repository structure

```
baseline-validation/
├── README.md                         # this file
├── ansible.cfg                       # collection and role paths
├── collections/requirements.yml      # ansible.platform, ansible.controller
├── playbooks/validate_baseline.yml   # entry point
├── roles/baseline_validation/
│   ├── defaults/main.yml             # baseline thresholds (config-as-code)
│   ├── meta/main.yml
│   └── tasks/
│       ├── main.yml                  # orchestration
│       ├── gather_ssh_facts.yml      # SSH service + effective sshd config (needs become)
│       ├── checks.yml                # data-driven evaluation loop
│       └── report.yml                # render report + gate on compliance
└── configure_aap/                    # stand up AAP objects for the demo
    ├── README.md
    ├── configure.yml
    └── vars/
        ├── organizations.yml
        ├── projects.yml
        ├── inventories.yml
        ├── credentials.yml           # credential object only — no secrets
        └── job_templates.yml         # job template + survey definition
```

## Quick start

### Step 1 — Install collections

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

### Step 2 — Stand up AAP objects

See [`configure_aap/README.md`](configure_aap/README.md) for full instructions.

```bash
export CONTROLLER_HOST=https://your-aap-controller.example.com
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD=yourpassword
ansible-playbook configure_aap/configure.yml
```

### Step 3 — Add secrets and hosts in AAP

1. Open **Credentials → Baseline Demo Machine Credential** and enter the SSH username and key/password for your target hosts. No secrets are stored in this repository.
2. Add your RHEL hosts to **Inventories → Baseline Demo Inventory**.

### Step 4 — Run the demo

1. Navigate to **Templates → Validate Baseline** and click **Launch**.
2. The survey lets you set threshold values live — raise a threshold above what the host has to force a NON-COMPLIANT result.
3. Watch the job output for the report.

## Changing the baseline

Default thresholds are in [`roles/baseline_validation/defaults/main.yml`](roles/baseline_validation/defaults/main.yml):

| Check | Default expectation | Variable |
|-------|--------------------|----------|
| OS version | RHEL 9.0 | `baseline_os_min_version` |
| CPU cores | 2 | `baseline_cpu_min_cores` |
| Memory | 4 GB | `baseline_mem_min_gb` |
| Root filesystem | 40 GB | `baseline_root_fs_min_gb` |
| SSH service | enabled at boot **and** running | `baseline_ssh_service_expected` |
| Password authentication disabled | `no` | `baseline_ssh_password_auth_expected` |
| Root login disabled | `no` (strict — `prohibit-password` fails) | `baseline_ssh_permit_root_expected` |

You can override them:
- **Permanently** — edit `defaults/main.yml` and commit.
- **Per job run** — use the AAP survey (no code change needed).

> **Privilege escalation required.** The SSH checks read the effective `sshd` config (`sshd -T`) and service state, which need root. The role runs those gather tasks with `become: true`, so the AAP **machine credential must permit privilege escalation** (e.g. passwordless sudo, or a sudo password supplied on the credential).

## Adding a new check

The check engine is data-driven. To add a new check, add one entry to the `baseline_checks` list in `roles/baseline_validation/defaults/main.yml`:

```yaml
- name: "Swap space"
  fact_expr: "(ansible_facts['swaptotal_mb'] / 1024) | round(1)"
  operator: "ge"
  threshold: "{{ baseline_swap_min_gb | default(2) }}"
  unit: "GB"
```

No changes to any task file are required.

Supported operators: `ge` (≥), `le` (≤), `eq` (=), `version_ge`, `version_eq`.

## Fail vs report-only mode

By default the job fails (red) when a host is non-compliant. Set `baseline_fail_on_noncompliance: false` to switch to report-only mode. This variable is also exposed on the survey so you can toggle it live during a demo.

## Testing locally

Run against `localhost` to verify the role works without a target RHEL host:

```bash
# Compliant run (defaults)
ansible-playbook playbooks/validate_baseline.yml -i localhost, \
  -e ansible_connection=local

# Force a failure by raising the memory threshold above what your machine has
ansible-playbook playbooks/validate_baseline.yml -i localhost, \
  -e ansible_connection=local \
  -e baseline_mem_min_gb=9999

# Report-only (no job failure)
ansible-playbook playbooks/validate_baseline.yml -i localhost, \
  -e ansible_connection=local \
  -e baseline_mem_min_gb=9999 \
  -e baseline_fail_on_noncompliance=false
```

## Lint

```bash
ansible-lint
yamllint .
```

## Collections used

| Collection | Purpose |
|-----------|---------|
| `ansible.builtin` | `setup`, `set_fact`, `debug`, `fail`, `import_tasks` |
| `ansible.platform` | AAP organization management in config-as-code |
| `ansible.controller` | AAP project, inventory, credential, job template management |

Certified and validated content only — no Galaxy community collections.
