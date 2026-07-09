# Baseline Validation

An Ansible Automation Platform (AAP) solution that validates whether a RHEL or Windows host meets a defined technical baseline and produces a clear **PASS / NON-COMPLIANT** report in the job output.

## What it does

When you run the **Validate Baseline - RHEL** or **Validate Baseline - Windows** job template against a target host, the playbook:

1. Gathers live facts from the target host, including SSH service state and effective `sshd` configuration.
2. Compares the baseline parameters — OS version, CPU cores, memory, root filesystem, and three SSH security controls — against configurable expectations.
3. Prints a report in the job output — every check, pass or fail, with the actual value found.
4. Fails the job (red in AAP) if any check fails, so you get an immediate visual signal.

The three SSH security controls can be bypassed at launch via the **Security** survey question — see [Bypassing the security checks](#bypassing-the-security-checks).

The role supports both RHEL and Windows hosts from the same inventory: run **Validate Baseline - RHEL** against your `rhel` group, and **Validate Baseline - Windows** against your `windows` group — each uses its own Machine credential and can be launched independently or in parallel.

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
├── collections/requirements.yml      # ansible.platform, ansible.controller, ansible.windows
├── playbooks/validate_baseline.yml   # entry point
├── roles/baseline_validation/
│   ├── defaults/main.yml             # RHEL + Windows thresholds and checks (config-as-code)
│   ├── meta/main.yml
│   └── tasks/
│       ├── main.yml                  # orchestration: branches by os_family
│       ├── rhel.yml                  # RHEL-specific fact derivation
│       ├── gather_ssh_facts.yml      # SSH service + effective sshd config (needs become)
│       ├── windows.yml               # Windows-specific fact derivation
│       ├── gather_windows_security_facts.yml  # Firewall/RDP/local admin (WinRM)
│       ├── checks.yml                # data-driven evaluation loop, platform-selected
│       └── report.yml                # render report + gate on compliance
└── configure_aap/                    # stand up AAP objects for the demo
    ├── README.md
    ├── configure.yml
    └── vars/
        ├── organizations.yml
        ├── projects.yml
        ├── inventories.yml
        ├── groups.yml                # rhel / windows inventory groups + WinRM connection vars
        ├── credentials.yml           # credential objects only — no secrets
        └── job_templates.yml         # two job templates (RHEL, Windows) + surveys
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

1. Open **Credentials → Baseline Demo Machine Credential** and enter the SSH username and key/password for your RHEL hosts.
2. Open **Credentials → Baseline Windows Machine Credential** and enter the username/password for your Windows hosts (CredSSP).
3. Add your RHEL hosts to the **rhel** group and your Windows hosts to the **windows** group in **Inventories → Baseline Demo Inventory**. No secrets are stored in this repository.

### Step 4 — Run the demo

1. Navigate to **Templates → Validate Baseline - RHEL** (or **Validate Baseline - Windows**) and click **Launch**.
2. The survey lets you set threshold values live — raise a threshold above what the host has to force a NON-COMPLIANT result.
3. The **Security** survey question toggles the platform's security checks (SSH hardening for RHEL; Firewall/RDP NLA/local Administrator for Windows) — select **no** to bypass them.
4. Watch the job output for the report. Both templates can be launched in parallel since they're scoped to disjoint groups.

## Changing the baseline

Default thresholds are in [`roles/baseline_validation/defaults/main.yml`](roles/baseline_validation/defaults/main.yml).

**RHEL** — checks list `baseline_checks_rhel`:

| Check | Default expectation | Variable |
|-------|--------------------|----------|
| OS version | RHEL 9.0 | `baseline_os_min_version` |
| CPU cores | 2 | `baseline_cpu_min_cores` |
| Memory | 4 GB | `baseline_mem_min_gb` |
| Root filesystem | 40 GB | `baseline_root_fs_min_gb` |
| SSH service | enabled at boot **and** running | `baseline_ssh_service_expected` |
| Password authentication disabled | `no` | `baseline_ssh_password_auth_expected` |
| Root login disabled | `no` (strict — `prohibit-password` fails) | `baseline_ssh_permit_root_expected` |

**Windows** — same file, checks list `baseline_checks_windows`:

| Check | Default expectation | Variable |
|-------|--------------------|----------|
| OS build | 10.0.20348 (Server 2022) | `baseline_win_os_min_build` |
| CPU cores | 2 | `baseline_cpu_min_cores` |
| Memory | 4 GB | `baseline_mem_min_gb` |
| C: drive capacity | 40 GB | `baseline_root_fs_min_gb` |
| Windows Firewall enabled (all profiles) | `yes` | `baseline_win_firewall_expected` |
| RDP Network Level Authentication | `yes` | `baseline_win_rdp_nla_expected` |
| Built-in Administrator account disabled | `yes` | `baseline_win_local_admin_disabled_expected` |

You can override them:
- **Permanently** — edit `defaults/main.yml` and commit.
- **Per job run** — use the AAP survey (no code change needed).

> **Privilege escalation required.** The SSH checks read the effective `sshd` config (`sshd -T`) and service state, which need root. The role runs those gather tasks with `become: true`, so the AAP **machine credential must permit privilege escalation** (e.g. passwordless sudo, or a sudo password supplied on the credential).

## Adding a new check

The check engine is data-driven. To add a new check, add one entry to `baseline_checks_rhel` or `baseline_checks_windows` (whichever platform it applies to) in `roles/baseline_validation/defaults/main.yml`:

```yaml
- name: "Swap space"
  category: "system"
  fact_expr: "(ansible_facts['swaptotal_mb'] / 1024) | round(1)"
  operator: "ge"
  threshold: "{{ baseline_swap_min_gb | default(2) }}"
  unit: "GB"
```

Set `category: "security"` for a check that should be skipped when the **Security** survey question is set to `no`; use `"system"` (or any other value) otherwise.

No changes to any task file are required.

Supported operators: `ge` (≥), `le` (≤), `eq` (=), `version_ge`, `version_eq`.

## Fail vs report-only mode

By default the job fails (red) when a host is non-compliant. Set `baseline_fail_on_noncompliance: false` to switch to report-only mode. This variable is also exposed on the survey so you can toggle it live during a demo.

## Bypassing the security checks

The three SSH security controls (SSH service, password authentication, root login) can be skipped entirely — useful when validating hosts where SSH facts can't be gathered, or where privilege escalation isn't available.

Set `baseline_security_checks_enabled: "no"` (default `"yes"`), exposed on the survey as the **Security** question. When disabled:

- The privileged SSH fact-gathering (`sshd -T`, service state — all `become: true`) is **skipped**, so no root access is needed.
- The three security checks are **filtered out of the report** rather than run and ignored — the report shows only the four system checks.

Each check carries a `category` (`system` or `security`); the bypass filters on `category: security`, so any security check you add later is covered automatically.

## Windows prerequisites

- Target Windows hosts must be reachable over WinRM (port 5986) with CredSSP authentication configured, and the connecting account needs local Administrator rights on the target — the Windows security checks (Firewall, RDP NLA, local Administrator account) run without privilege escalation and assume the WinRM session is already elevated.
- The `windows` inventory group's connection variables (`ansible_connection`, `ansible_winrm_transport`, etc.) are set as AAP group variables by `configure_aap/vars/groups.yml` — no per-host overrides needed.

## Testing locally

Run against `localhost` to verify the RHEL path works without a target RHEL host. **There is no equivalent local-testing trick for Windows** — WinRM requires a real reachable Windows target, so the Windows checks must be exercised against a real or test Windows host reachable over WinRM/CredSSP.

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

# Bypass the SSH security checks (system checks only)
ansible-playbook playbooks/validate_baseline.yml -i localhost, \
  -e ansible_connection=local \
  -e baseline_security_checks_enabled=no
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
| `ansible.windows` | `setup`, `win_shell` for Windows fact gathering |

Certified and validated content only — no Galaxy community collections.
