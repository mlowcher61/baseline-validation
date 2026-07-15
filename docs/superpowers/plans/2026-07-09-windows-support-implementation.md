# Windows Baseline Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the `baseline_validation` role and its AAP config-as-code to validate Windows hosts alongside RHEL hosts, in the same repository, via two parallel job templates each scoped to its own inventory group and credential.

**Architecture:** One role branches on `ansible_facts['os_family']` for fact-gathering (`tasks/rhel.yml` vs `tasks/windows.yml`) but shares one evaluation engine (`checks.yml`) and one report renderer (`report.yml`), each now selecting between `baseline_checks_rhel` and `baseline_checks_windows`. Two AAP job templates ("Validate Baseline - RHEL", "Validate Baseline - Windows") run against the same inventory, each `limit`-scoped to its own group (`rhel`, `windows`) with its own Machine credential.

**Tech Stack:** Ansible Automation Platform, `ansible.builtin`, `ansible.windows` (new), `ansible.platform`, `ansible.controller`.

## Global Constraints

- Built for Ansible Automation Platform, not ansible-core.
- Certified/validated content only — no Galaxy community collections.
- Prefer `ansible.platform` over `ansible.controller` where the resource is covered by it; controller-specific resources (inventory, group, credential, job template, survey) stay on `ansible.controller`, matching the existing pattern in `configure_aap/configure.yml`.
- No secrets in git — credential objects are config-as-code, secret values are set in the AAP UI.
- Config-as-code for all new AAP objects (groups, credential, job template) so a new user can stand up the whole solution from git.
- README(s) must fully explain the solution for a new Ansible user.
- Reference design spec: `docs/superpowers/specs/2026-07-09-windows-support-design.md`.

---

### Task 1: Collections, role metadata, and play-level fact gathering

**Files:**
- Modify: `collections/requirements.yml`
- Modify: `roles/baseline_validation/meta/main.yml`
- Modify: `playbooks/validate_baseline.yml`

**Interfaces:**
- Consumes: nothing (first task).
- Produces: `ansible.windows` collection available for later tasks; facts are gathered implicitly at play level (correct module auto-selected per connection), so `ansible_facts['os_family']` is populated before the role runs.

- [ ] **Step 1: Add the `ansible.windows` collection**

Edit `collections/requirements.yml` to:

```yaml
---
collections:
  - name: ansible.platform
  - name: ansible.controller
  - name: ansible.windows
```

- [ ] **Step 2: Install it and verify**

Run: `ansible-galaxy collection install -r collections/requirements.yml`
Expected: `ansible.windows` installs alongside the existing two collections, no errors.

- [ ] **Step 3: Update role metadata for Windows**

Edit `roles/baseline_validation/meta/main.yml` to:

```yaml
---
galaxy_info:
  author: mlowcher61
  description: Validates a RHEL or Windows host against a defined technical baseline
  license: MIT
  min_ansible_version: "2.14"
  platforms:
    - name: EL
      versions:
        - "8"
        - "9"
    - name: Windows
      versions:
        - all
  galaxy_tags:
    - baseline
    - compliance
    - rhel
    - windows
dependencies: []
```

- [ ] **Step 4: Switch to play-level fact gathering**

`ansible.builtin.setup` only works over SSH. To support both platforms, let Ansible auto-select the facts module (`ansible.builtin.setup` for SSH, `ansible.windows.setup` for WinRM) based on each host's connection. Edit `playbooks/validate_baseline.yml`:

```yaml
---
- name: Validate host against technical baseline
  hosts: all
  gather_facts: true

  roles:
    - role: baseline_validation
```

(This changes `gather_facts: false` to `true`. `roles/baseline_validation/tasks/main.yml` still has its own explicit `ansible.builtin.setup:` task at this point — that's fine, it just re-gathers facts redundantly until Task 3 removes it. Don't remove it yet; Task 3 restructures that file.)

- [ ] **Step 5: Regression check against the current RHEL path**

Run:
```bash
ansible-playbook playbooks/validate_baseline.yml -i localhost, -e ansible_connection=local
```
Expected: Output still shows the `BASELINE VALIDATION — localhost` report with all four system checks and (if run as a user that can sudo) the three SSH checks, exactly as before this task. This confirms `gather_facts: true` plus the now-redundant explicit `setup` task doesn't break anything.

- [ ] **Step 6: Commit**

```bash
git add collections/requirements.yml roles/baseline_validation/meta/main.yml playbooks/validate_baseline.yml
git commit -m "Add ansible.windows collection and switch to play-level fact gathering"
```

---

### Task 2: Restructure defaults for dual RHEL/Windows checks lists

**Files:**
- Modify: `roles/baseline_validation/defaults/main.yml`

**Interfaces:**
- Consumes: nothing new.
- Produces: `baseline_checks_rhel`, `baseline_checks_windows` (lists consumed by Task 5's `checks.yml`); `baseline_win_os_min_build`, `baseline_win_firewall_expected`, `baseline_win_rdp_nla_expected`, `baseline_win_local_admin_disabled_expected` (survey-overridable thresholds, consumed by Task 7's Windows survey); `bv_win_firewall_all_enabled`, `bv_win_rdp_nla_enabled`, `bv_win_local_admin_enabled` (safe-default facts, overridden by Task 4's `gather_windows_security_facts.yml`).

- [ ] **Step 1: Rewrite `defaults/main.yml`**

Replace the entire file with:

```yaml
---
# Baseline thresholds — all overridable via AAP survey variables

# --- RHEL ---
baseline_os_min_version: "9.0"
baseline_cpu_min_cores: 2
baseline_mem_min_gb: 4
baseline_root_fs_min_gb: 40

# Security baseline expectations (override via survey if needed)
baseline_ssh_service_expected: "yes"        # sshd enabled at boot AND running
baseline_ssh_password_auth_expected: "no"   # PasswordAuthentication disabled
baseline_ssh_permit_root_expected: "no"     # PermitRootLogin strictly no

# Set to false to report only without failing the job
baseline_fail_on_noncompliance: true

# Set to "no" (via the survey) to bypass the security baseline checks entirely,
# for both RHEL (SSH) and Windows (Firewall/RDP/local admin).
baseline_security_checks_enabled: "yes"

# Safe defaults for the SSH facts gathered by tasks/gather_ssh_facts.yml. These must be
# defined even when that task is skipped (security bypassed): Ansible templates the
# entire checks list — including the security checks' fact_expr — whenever the
# evaluation loop runs, so the referenced vars cannot be undefined. When security
# is enabled, gather_ssh_facts.yml set_fact overrides these with the real values.
bv_ssh_enabled: false
bv_ssh_active: false
bv_sshd_password_auth: "unknown"
bv_sshd_permit_root: "unknown"

# --- Windows ---
baseline_win_os_min_build: "10.0.20348"     # Windows Server 2022
baseline_win_firewall_expected: "yes"       # all firewall profiles enabled
baseline_win_rdp_nla_expected: "yes"        # RDP Network Level Authentication enforced
baseline_win_local_admin_disabled_expected: "yes"   # built-in Administrator account disabled

# Safe defaults for the Windows facts gathered by tasks/gather_windows_security_facts.yml.
# Same "must always be defined" rule as the bv_ssh_* facts above.
bv_win_firewall_all_enabled: false
bv_win_rdp_nla_enabled: false
bv_win_local_admin_enabled: true

# Data-driven check definitions — add new checks here, no logic changes needed.
# `category` tags a check so groups (e.g. security) can be filtered as a set.
baseline_checks_rhel:
  - name: "OS version"
    category: "system"
    fact_expr: "{{ ansible_facts['distribution_version'] }}"
    operator: "version_ge"
    threshold: "{{ baseline_os_min_version }}"
    unit: ""

  - name: "CPU cores"
    category: "system"
    fact_expr: "{{ ansible_facts['processor_vcpus'] | int }}"
    operator: "ge"
    threshold: "{{ baseline_cpu_min_cores | int }}"
    unit: "cores"

  - name: "Memory"
    category: "system"
    fact_expr: "{{ (ansible_facts['memtotal_mb'] / 1024) | round(1) }}"
    operator: "ge"
    threshold: "{{ baseline_mem_min_gb | float }}"
    unit: "GB"

  - name: "Root filesystem capacity"
    category: "system"
    fact_expr: "{{ root_fs_size_gb }}"
    operator: "ge"
    threshold: "{{ baseline_root_fs_min_gb | int }}"
    unit: "GB"

  - name: "SSH service"
    category: "security"
    fact_expr: "{{ 'yes' if (bv_ssh_enabled | bool and bv_ssh_active | bool) else 'no' }}"
    operator: "eq"
    threshold: "{{ baseline_ssh_service_expected }}"
    unit: "(enabled+running)"

  - name: "Password authentication disabled"
    category: "security"
    fact_expr: "{{ bv_sshd_password_auth }}"
    operator: "eq"
    threshold: "{{ baseline_ssh_password_auth_expected }}"
    unit: ""

  - name: "Root login disabled"
    category: "security"
    fact_expr: "{{ bv_sshd_permit_root }}"
    operator: "eq"
    threshold: "{{ baseline_ssh_permit_root_expected }}"
    unit: ""

baseline_checks_windows:
  - name: "OS build"
    category: "system"
    fact_expr: "{{ ansible_facts['distribution_version'] }}"
    operator: "version_ge"
    threshold: "{{ baseline_win_os_min_build }}"
    unit: ""

  - name: "CPU cores"
    category: "system"
    fact_expr: "{{ ansible_facts['processor_vcpus'] | int }}"
    operator: "ge"
    threshold: "{{ baseline_cpu_min_cores | int }}"
    unit: "cores"

  - name: "Memory"
    category: "system"
    fact_expr: "{{ (ansible_facts['memtotal_mb'] / 1024) | round(1) }}"
    operator: "ge"
    threshold: "{{ baseline_mem_min_gb | float }}"
    unit: "GB"

  - name: "C: drive capacity"
    category: "system"
    fact_expr: "{{ win_c_drive_size_gb }}"
    operator: "ge"
    threshold: "{{ baseline_root_fs_min_gb | int }}"
    unit: "GB"

  - name: "Windows Firewall enabled"
    category: "security"
    fact_expr: "{{ 'yes' if bv_win_firewall_all_enabled | bool else 'no' }}"
    operator: "eq"
    threshold: "{{ baseline_win_firewall_expected }}"
    unit: "(all profiles)"

  - name: "RDP Network Level Authentication"
    category: "security"
    fact_expr: "{{ 'yes' if bv_win_rdp_nla_enabled | bool else 'no' }}"
    operator: "eq"
    threshold: "{{ baseline_win_rdp_nla_expected }}"
    unit: ""

  - name: "Built-in Administrator account disabled"
    category: "security"
    fact_expr: "{{ 'yes' if not (bv_win_local_admin_enabled | bool) else 'no' }}"
    operator: "eq"
    threshold: "{{ baseline_win_local_admin_disabled_expected }}"
    unit: ""
```

- [ ] **Step 2: Verify YAML is well-formed**

Run: `yamllint roles/baseline_validation/defaults/main.yml`
Expected: no errors.

Note: `ansible-playbook` will now fail if run at this point, because `checks.yml` (Task 5) and `rhel.yml`/`windows.yml` (Tasks 3–4) still reference the old `baseline_checks` name and `root_fs_size_gb`/`win_c_drive_size_gb` aren't wired up yet. That's expected — this task only lands the data, later tasks wire the consumers. Do not run the full playbook yet.

- [ ] **Step 3: Commit**

```bash
git add roles/baseline_validation/defaults/main.yml
git commit -m "Split baseline_checks into baseline_checks_rhel and baseline_checks_windows"
```

---

### Task 3: Split RHEL fact-gathering into its own task file

**Files:**
- Create: `roles/baseline_validation/tasks/rhel.yml`
- Modify: `roles/baseline_validation/tasks/main.yml`
- Modify: `roles/baseline_validation/tasks/gather_ssh_facts.yml` (no content change — confirm it still fits under the new orchestration)

**Interfaces:**
- Consumes: `ansible_facts['os_family']` (from Task 1's play-level fact gathering), `baseline_security_checks_enabled` (from Task 2).
- Produces: `root_fs_size_gb` (consumed by `baseline_checks_rhel`'s "Root filesystem capacity" check), `bv_ssh_*` facts (consumed by the RHEL security checks) — same as before, just relocated.

- [ ] **Step 1: Create `tasks/rhel.yml`**

This is the RHEL-specific portion of what used to be in `tasks/main.yml` (minus the `ansible.builtin.setup` call, which is now handled at the play level per Task 1):

```yaml
---
- name: Derive root filesystem size in GB
  ansible.builtin.set_fact:
    root_fs_size_gb: >-
      {{
        (ansible_facts['mounts']
          | selectattr('mount', 'equalto', '/')
          | map(attribute='size_total')
          | first
          | int / 1073741824
        ) | round(1)
      }}

- name: Gather SSH security facts
  ansible.builtin.import_tasks: gather_ssh_facts.yml
  when: baseline_security_checks_enabled | bool
```

- [ ] **Step 2: Rewrite `tasks/main.yml` to branch by OS family**

```yaml
---
- name: Include RHEL fact-gathering
  ansible.builtin.import_tasks: rhel.yml
  when: ansible_facts['os_family'] == 'RedHat'

- name: Include Windows fact-gathering
  ansible.builtin.import_tasks: windows.yml
  when: ansible_facts['os_family'] == 'Windows'

- name: Evaluate baseline checks
  ansible.builtin.import_tasks: checks.yml

- name: Render report and gate on compliance
  ansible.builtin.import_tasks: report.yml
```

(`windows.yml` doesn't exist yet — that's Task 4. The `when: os_family == 'Windows'` branch will simply not trigger on the RHEL smoke test below, so this is safe to land now.)

- [ ] **Step 3: Regression check**

Run:
```bash
ansible-playbook playbooks/validate_baseline.yml -i localhost, -e ansible_connection=local
```
Expected: Same report as Task 1's Step 5 — the RHEL path is unchanged in behavior, just relocated into `rhel.yml`.

- [ ] **Step 4: Commit**

```bash
git add roles/baseline_validation/tasks/rhel.yml roles/baseline_validation/tasks/main.yml
git commit -m "Split RHEL fact-gathering into tasks/rhel.yml, branch main.yml by os_family"
```

---

### Task 4: Windows fact-gathering

**Files:**
- Create: `roles/baseline_validation/tasks/windows.yml`
- Create: `roles/baseline_validation/tasks/gather_windows_security_facts.yml`

**Interfaces:**
- Consumes: `ansible_facts['mounts']` (Windows facts, assumed to carry drive-letter mount points — verify per Step 4 below), `baseline_security_checks_enabled` (from Task 2).
- Produces: `win_c_drive_size_gb`, `bv_win_firewall_all_enabled`, `bv_win_rdp_nla_enabled`, `bv_win_local_admin_enabled` — consumed by `baseline_checks_windows` in `defaults/main.yml`.

- [ ] **Step 1: Create `tasks/windows.yml`**

```yaml
---
- name: Derive C: drive capacity in GB
  ansible.builtin.set_fact:
    win_c_drive_size_gb: >-
      {{
        (ansible_facts['mounts']
          | selectattr('mount', 'equalto', 'C:')
          | map(attribute='size_total')
          | first
          | int / 1073741824
        ) | round(1)
      }}

- name: Gather Windows security facts
  ansible.builtin.import_tasks: gather_windows_security_facts.yml
  when: baseline_security_checks_enabled | bool
```

- [ ] **Step 2: Create `tasks/gather_windows_security_facts.yml`**

```yaml
---
# Gather Windows security signals that are NOT present in standard ansible_facts.
# Uses PowerShell via ansible.windows.win_shell. No become/runas — this assumes
# the WinRM account already holds local admin rights on the target (see README
# "Windows prerequisites" section). If that assumption doesn't hold in your
# environment, add `become: true` / `become_method: runas` to the three
# win_shell tasks below.

- name: Query Windows Firewall profile state
  ansible.windows.win_shell: |
    Get-NetFirewallProfile | Select-Object Name, Enabled | ConvertTo-Json
  register: __bv_win_firewall
  changed_when: false

- name: Derive firewall-all-profiles-enabled fact
  ansible.builtin.set_fact:
    bv_win_firewall_all_enabled: "{{ (__bv_win_firewall.stdout | from_json) | map(attribute='Enabled') | select | list | length == 3 }}"

- name: Query RDP Network Level Authentication setting
  ansible.windows.win_shell: |
    (Get-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name UserAuthentication).UserAuthentication
  register: __bv_win_rdp_nla
  changed_when: false

- name: Derive RDP NLA fact
  ansible.builtin.set_fact:
    bv_win_rdp_nla_enabled: "{{ __bv_win_rdp_nla.stdout | trim == '1' }}"

- name: Query built-in Administrator account state
  ansible.windows.win_shell: |
    Get-LocalUser | Where-Object { $_.SID -like '*-500' } | Select-Object Enabled | ConvertTo-Json
  register: __bv_win_local_admin
  changed_when: false

- name: Derive local Administrator account fact
  ansible.builtin.set_fact:
    bv_win_local_admin_enabled: "{{ (__bv_win_local_admin.stdout | from_json).Enabled | bool }}"
```

- [ ] **Step 3: Syntax-check (no Windows host available in this environment)**

Run: `ansible-playbook playbooks/validate_baseline.yml --syntax-check`
Expected: `playbook: playbooks/validate_baseline.yml` with no errors. This confirms the YAML/Jinja is valid; it does **not** prove the PowerShell or fact-key assumptions below are correct — that requires Step 4.

- [ ] **Step 4: Verify fact-key assumptions against a real Windows host**

This design assumes `ansible.windows.setup` populates `ansible_facts['processor_vcpus']`, `ansible_facts['memtotal_mb']`, `ansible_facts['distribution_version']`, and `ansible_facts['mounts']` (with `mount: 'C:'`) the same way `ansible.builtin.setup` does on RHEL. Confirm this against a real WinRM-reachable Windows host before relying on it in a demo:

```bash
ansible windows_test_host -i <your_inventory> -m ansible.windows.setup -a "filter=ansible_processor_vcpus,ansible_memtotal_mb,ansible_distribution_version,ansible_mounts"
```

Expected: all four keys appear in the output with non-empty values, and `ansible_mounts` contains an entry with `"mount": "C:"` and a `size_total` in bytes.

If any key name differs (e.g. `ansible_facts['mounts']` isn't populated for Windows in your collection version), adjust the corresponding `fact_expr` in `defaults/main.yml` (Task 2) and the `win_c_drive_size_gb` derivation above accordingly — for example, some environments may need `ansible.windows.win_shell: (Get-Volume -DriveLetter C).Size` as a fallback for the C: drive size instead of `ansible_facts['mounts']`.

Also verify the three PowerShell one-liners in `gather_windows_security_facts.yml` directly on the test host (e.g. via `ansible windows_test_host -m ansible.windows.win_shell -a "Get-NetFirewallProfile | Select-Object Name, Enabled | ConvertTo-Json"`) to confirm the JSON shape matches what `from_json` / `map(attribute=...)` expects above.

- [ ] **Step 5: Commit**

```bash
git add roles/baseline_validation/tasks/windows.yml roles/baseline_validation/tasks/gather_windows_security_facts.yml
git commit -m "Add Windows fact-gathering (system + security checks)"
```

---

### Task 5: Select the active checks list by OS family

**Files:**
- Modify: `roles/baseline_validation/tasks/checks.yml`

**Interfaces:**
- Consumes: `baseline_checks_rhel`, `baseline_checks_windows` (Task 2), `ansible_facts['os_family']`.
- Produces: `baseline_results` (unchanged shape — consumed by `report.yml`, which needs no changes).

- [ ] **Step 1: Add a platform-selection task and swap the loop source**

Edit `roles/baseline_validation/tasks/checks.yml` to:

```yaml
---
# Evaluate every check and collect results; never short-circuit on first failure.
- name: Select platform checks list
  ansible.builtin.set_fact:
    __bv_active_checks: "{{ baseline_checks_windows if ansible_facts['os_family'] == 'Windows' else baseline_checks_rhel }}"

- name: Evaluate check — {{ item.name }}
  ansible.builtin.set_fact:
    baseline_results: >-
      {{
        baseline_results | default([]) + [{
          'name':      item.name,
          'found':     lookup('vars', '__bv_found') | string,
          'threshold': item.threshold | string,
          'unit':      item.unit,
          'operator':  item.operator,
          'pass':      __bv_pass
        }]
      }}
  vars:
    __bv_found: "{{ item.fact_expr | trim }}"
    __bv_pass: >-
      {%- if item.operator == 'version_ge' -%}
        {{ __bv_found is version(item.threshold, '>=') }}
      {%- elif item.operator == 'ge' -%}
        {{ __bv_found | float >= item.threshold | float }}
      {%- elif item.operator == 'le' -%}
        {{ __bv_found | float <= item.threshold | float }}
      {%- elif item.operator == 'eq' -%}
        {{ __bv_found == item.threshold | string }}
      {%- elif item.operator == 'version_eq' -%}
        {{ __bv_found is version(item.threshold, '==') }}
      {%- else -%}
        false
      {%- endif -%}
  loop: >-
    {{ __bv_active_checks if (baseline_security_checks_enabled | bool)
       else (__bv_active_checks | rejectattr('category', 'equalto', 'security') | list) }}
  loop_control:
    label: "{{ item.name }}"
```

Only two things changed from the original: the new "Select platform checks list" task, and `baseline_checks` → `__bv_active_checks` in the `loop:` line.

- [ ] **Step 2: Full RHEL regression pass**

Run each of these and check the described output — this is the same test matrix documented in the design spec §8:

```bash
# Compliant run (defaults)
ansible-playbook playbooks/validate_baseline.yml -i localhost, -e ansible_connection=local
```
Expected: report renders, `RESULT: COMPLIANT` (or `NON-COMPLIANT` only on genuinely failing checks like SSH hardening, same as before this change set).

```bash
# Force a failure
ansible-playbook playbooks/validate_baseline.yml -i localhost, -e ansible_connection=local -e baseline_mem_min_gb=9999
```
Expected: `[ FAIL ] Memory` line present, `RESULT: NON-COMPLIANT`, job exits non-zero.

```bash
# Report-only
ansible-playbook playbooks/validate_baseline.yml -i localhost, -e ansible_connection=local -e baseline_mem_min_gb=9999 -e baseline_fail_on_noncompliance=false
```
Expected: same NON-COMPLIANT report, but the play does not fail (exit 0).

```bash
# Bypass security checks
ansible-playbook playbooks/validate_baseline.yml -i localhost, -e ansible_connection=local -e baseline_security_checks_enabled=no
```
Expected: report shows only the four system checks, no SSH checks, no privilege escalation attempted.

- [ ] **Step 3: Lint**

Run: `ansible-lint roles/baseline_validation` and `yamllint roles/baseline_validation`
Expected: clean (or same pre-existing findings as before this change set, if any).

- [ ] **Step 4: Commit**

```bash
git add roles/baseline_validation/tasks/checks.yml
git commit -m "Select baseline_checks_rhel or baseline_checks_windows by os_family"
```

---

### Task 6: Config-as-code — inventory groups and Windows credential

**Files:**
- Create: `configure_aap/vars/groups.yml`
- Modify: `configure_aap/vars/credentials.yml`
- Modify: `configure_aap/configure.yml`

**Interfaces:**
- Consumes: `aap_inventories` (existing, from `inventories.yml`).
- Produces: `aap_groups`, extended `aap_credentials` — consumed by Task 7's `job_templates.yml` (`credentials:` list references credential names by string, no direct variable coupling).

- [ ] **Step 1: Create `configure_aap/vars/groups.yml`**

```yaml
---
# Inventory groups for Baseline Demo Inventory. Hosts are added manually in
# the AAP UI (or via a dynamic source) — these entries create the group and,
# for Windows, set the WinRM/CredSSP connection variables so hosts in that
# group don't need per-host connection overrides.
aap_groups:
  - name: "rhel"
    inventory: "Baseline Demo Inventory"
    description: "RHEL hosts validated by the Validate Baseline - RHEL job template"

  - name: "windows"
    inventory: "Baseline Demo Inventory"
    description: "Windows hosts validated by the Validate Baseline - Windows job template"
    variables:
      ansible_connection: winrm
      ansible_winrm_transport: credssp
      ansible_port: 5986
      ansible_winrm_server_cert_validation: ignore
```

- [ ] **Step 2: Add the Windows credential to `configure_aap/vars/credentials.yml`**

```yaml
---
# No secrets stored in git. The credential object is defined here;
# supply the actual username/password/SSH key in AAP at apply time.
aap_credentials:
  - name: "Baseline Demo Machine Credential"
    organization: "Demo"
    credential_type: "Machine"
    description: "SSH credential for target RHEL hosts — secret values set in AAP, not git"

  - name: "Baseline Windows Machine Credential"
    organization: "Demo"
    credential_type: "Machine"
    description: "CredSSP credential for target Windows hosts — secret values set in AAP, not git"
```

- [ ] **Step 3: Wire the group creation into `configure_aap/configure.yml`**

Edit the `vars_files:` block to add `vars/groups.yml` after `vars/inventories.yml`:

```yaml
  vars_files:
    - vars/organizations.yml
    #- vars/projects.yml # Manually created the baseline-validation project in the AAP
    - vars/inventories.yml
    - vars/groups.yml
    - vars/credentials.yml
    - vars/job_templates.yml
```

Add a new task immediately after the existing "Create inventories" task:

```yaml
    - name: Create inventory groups
      ansible.controller.group:
        name: "{{ item.name }}"
        inventory: "{{ item.inventory }}"
        description: "{{ item.description | default(omit) }}"
        variables: "{{ item.variables | default(omit) }}"
        state: present
      loop: "{{ aap_groups }}"
```

- [ ] **Step 4: Syntax-check**

Run: `ansible-playbook configure_aap/configure.yml --syntax-check`
Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add configure_aap/vars/groups.yml configure_aap/vars/credentials.yml configure_aap/configure.yml
git commit -m "Add rhel/windows inventory groups and Windows machine credential to config-as-code"
```

---

### Task 7: Config-as-code — Windows job template and survey

**Files:**
- Modify: `configure_aap/vars/job_templates.yml`

**Interfaces:**
- Consumes: `baseline_win_os_min_build`, `baseline_win_firewall_expected`, `baseline_win_rdp_nla_expected`, `baseline_win_local_admin_disabled_expected` (Task 2, referenced by name as survey `variable:` values — no direct code coupling, just string matching).

- [ ] **Step 1: Rename the existing template and add `limit: rhel`**

In `configure_aap/vars/job_templates.yml`, change the first template's `name` from `"Validate Baseline"` to `"Validate Baseline - RHEL"`, and add a `limit: "rhel"` key (immediately after `credentials:`):

```yaml
  - name: "Validate Baseline - RHEL"
    organization: "Default"
    project: "Baseline Validation"
    playbook: "playbooks/validate_baseline.yml"
    inventory: "Baseline Demo Inventory"
    credentials:
      - "Baseline Demo Machine Credential"
    limit: "rhel"
    ask_variables_on_launch: false
    survey_enabled: true
    survey_spec:
      name: "Baseline Validation Survey"
      description: "Override default baseline thresholds live for demo purposes"
      spec:
        - question_name: "Minimum OS version (e.g. 9.0)"
          question_description: "Minimum RHEL major.minor version required"
          variable: baseline_os_min_version
          type: text
          default: "9.0"
          required: true

        - question_name: "Minimum CPU cores"
          question_description: "Minimum number of virtual CPU cores required"
          variable: baseline_cpu_min_cores
          type: integer
          default: 2
          min: 1
          required: true

        - question_name: "Minimum memory (GB)"
          question_description: "Minimum total RAM in gigabytes"
          variable: baseline_mem_min_gb
          type: integer
          default: 4
          min: 1
          required: true

        - question_name: "Minimum root filesystem (GB)"
          question_description: "Minimum root filesystem capacity in gigabytes"
          variable: baseline_root_fs_min_gb
          type: integer
          default: 40
          min: 1
          required: true

        - question_name: "Fail job on non-compliance?"
          question_description: "Set to true to fail the job when checks fail; false for report-only"
          variable: baseline_fail_on_noncompliance
          type: multiplechoice
          default: "true"
          choices:
            - "true"
            - "false"
          required: true

        - question_name: "Security"
          question_description: "Run the SSH security baseline checks? Select no to bypass them"
          variable: baseline_security_checks_enabled
          type: multiplechoice
          default: "yes"
          choices:
            - "yes"
            - "no"
          required: true
```

(Only `name` and the new `limit` line changed from the current file — the survey is untouched.)

- [ ] **Step 2: Add the Windows job template**

Append a second entry to `aap_job_templates`:

```yaml
  - name: "Validate Baseline - Windows"
    organization: "Default"
    project: "Baseline Validation"
    playbook: "playbooks/validate_baseline.yml"
    inventory: "Baseline Demo Inventory"
    credentials:
      - "Baseline Windows Machine Credential"
    limit: "windows"
    ask_variables_on_launch: false
    survey_enabled: true
    survey_spec:
      name: "Baseline Validation Survey - Windows"
      description: "Override default Windows baseline thresholds live for demo purposes"
      spec:
        - question_name: "Minimum OS build (e.g. 10.0.20348)"
          question_description: "Minimum Windows build number required"
          variable: baseline_win_os_min_build
          type: text
          default: "10.0.20348"
          required: true

        - question_name: "Minimum CPU cores"
          question_description: "Minimum number of virtual CPU cores required"
          variable: baseline_cpu_min_cores
          type: integer
          default: 2
          min: 1
          required: true

        - question_name: "Minimum memory (GB)"
          question_description: "Minimum total RAM in gigabytes"
          variable: baseline_mem_min_gb
          type: integer
          default: 4
          min: 1
          required: true

        - question_name: "Minimum C: drive capacity (GB)"
          question_description: "Minimum C: drive capacity in gigabytes"
          variable: baseline_root_fs_min_gb
          type: integer
          default: 40
          min: 1
          required: true

        - question_name: "Fail job on non-compliance?"
          question_description: "Set to true to fail the job when checks fail; false for report-only"
          variable: baseline_fail_on_noncompliance
          type: multiplechoice
          default: "true"
          choices:
            - "true"
            - "false"
          required: true

        - question_name: "Security"
          question_description: "Run the Windows security baseline checks (Firewall, RDP NLA, local Administrator)? Select no to bypass them"
          variable: baseline_security_checks_enabled
          type: multiplechoice
          default: "yes"
          choices:
            - "yes"
            - "no"
          required: true
```

- [ ] **Step 3: Syntax-check and lint**

Run: `ansible-playbook configure_aap/configure.yml --syntax-check && yamllint configure_aap/vars/job_templates.yml`
Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add configure_aap/vars/job_templates.yml
git commit -m "Rename RHEL job template, add limit scoping, and add Windows job template + survey"
```

---

### Task 8: Update documentation

**Files:**
- Modify: `README.md`
- Modify: `configure_aap/README.md`

**Interfaces:**
- Consumes: nothing (docs only).
- Produces: nothing (docs only).

- [ ] **Step 1: Update the top-level `README.md`**

Make these changes:

1. In "What it does", after the existing paragraph, add:

```markdown
The role supports both RHEL and Windows hosts from the same inventory: run
**Validate Baseline - RHEL** against your `rhel` group, and **Validate
Baseline - Windows** against your `windows` group — each uses its own
Machine credential and can be launched independently or in parallel.
```

2. Replace the "Repository structure" code block with:

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

3. In "Quick start", Step 3, replace with:

```markdown
### Step 3 — Add secrets and hosts in AAP

1. Open **Credentials → Baseline Demo Machine Credential** and enter the SSH username and key/password for your RHEL hosts.
2. Open **Credentials → Baseline Windows Machine Credential** and enter the username/password for your Windows hosts (CredSSP).
3. Add your RHEL hosts to the **rhel** group and your Windows hosts to the **windows** group in **Inventories → Baseline Demo Inventory**. No secrets are stored in this repository.
```

4. In "Quick start", Step 4, replace with:

```markdown
### Step 4 — Run the demo

1. Navigate to **Templates → Validate Baseline - RHEL** (or **Validate Baseline - Windows**) and click **Launch**.
2. The survey lets you set threshold values live — raise a threshold above what the host has to force a NON-COMPLIANT result.
3. The **Security** survey question toggles the platform's security checks (SSH hardening for RHEL; Firewall/RDP NLA/local Administrator for Windows) — select **no** to bypass them.
4. Watch the job output for the report. Both templates can be launched in parallel since they're scoped to disjoint groups.
```

5. In "Changing the baseline", replace the single table with two tables:

```markdown
**RHEL** — defaults in [`roles/baseline_validation/defaults/main.yml`](roles/baseline_validation/defaults/main.yml), checks list `baseline_checks_rhel`:

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
```

6. In "Adding a new check", note that a check must be added to the list matching its platform:

```markdown
The check engine is data-driven. To add a new check, add one entry to
`baseline_checks_rhel` or `baseline_checks_windows` (whichever platform it
applies to) in `roles/baseline_validation/defaults/main.yml`:
```

(keep the existing example and "Set `category`..." paragraph below it unchanged)

7. Add a new section after "Bypassing the security checks":

```markdown
## Windows prerequisites

- Target Windows hosts must be reachable over WinRM (port 5986) with CredSSP
  authentication configured, and the connecting account needs local
  Administrator rights on the target — the Windows security checks
  (Firewall, RDP NLA, local Administrator account) run without
  privilege escalation and assume the WinRM session is already elevated.
- The `windows` inventory group's connection variables
  (`ansible_connection`, `ansible_winrm_transport`, etc.) are set as AAP
  group variables by `configure_aap/vars/groups.yml` — no per-host
  overrides needed.

## Testing locally

Run against `localhost` to verify the RHEL path works without a target RHEL
host — the existing commands below are unchanged. **There is no equivalent
local-testing trick for Windows**: WinRM requires a real reachable Windows
target, so the Windows checks must be exercised against a real or test
Windows host reachable over WinRM/CredSSP.
```

(keep the existing "Testing locally" code examples where they are, just add the note above them)

8. In "Collections used", add a row:

```markdown
| `ansible.windows` | `setup`, `win_shell` for Windows fact gathering |
```

- [ ] **Step 2: Update `configure_aap/README.md`**

1. In the "What it creates" table, update the Job Template row and add rows:

```markdown
| Object | Name |
|--------|------|
| Organization | Demo |
| Project | Baseline Validation |
| Inventory | Baseline Demo Inventory |
| Inventory Groups | `rhel`, `windows` (Windows group carries WinRM/CredSSP connection vars) |
| Machine Credential | Baseline Demo Machine Credential (RHEL), Baseline Windows Machine Credential |
| Job Templates + Surveys | Validate Baseline - RHEL, Validate Baseline - Windows |
```

2. In the "Prerequisites" numbered list, update step 3 and 4:

```markdown
3. After `configure.yml` runs, open the AAP UI and add the actual SSH username/key to the **Baseline Demo Machine Credential**, and the CredSSP username/password to the **Baseline Windows Machine Credential** — the playbook creates the credential objects but cannot store secrets.

4. Add your target RHEL hosts to the **rhel** group and Windows hosts to the **windows** group within the **Baseline Demo Inventory** in the AAP UI (or wire up a dynamic inventory source).
```

- [ ] **Step 3: Review the diff**

Run: `git diff README.md configure_aap/README.md`
Expected: changes match the content above; no stray unrelated edits.

- [ ] **Step 4: Commit**

```bash
git add README.md configure_aap/README.md
git commit -m "Document Windows support in README and configure_aap README"
```

---

## Final verification (after all tasks)

- [ ] Run `ansible-lint .` and `yamllint .` from the repo root — clean.
- [ ] Run `ansible-playbook playbooks/validate_baseline.yml --syntax-check` and `ansible-playbook configure_aap/configure.yml --syntax-check` — both clean.
- [ ] Re-run the four RHEL smoke-test commands from Task 5, Step 2 — all still behave as documented.
- [ ] Confirm Task 4, Step 4's Windows fact-key verification was actually performed against a real host (not just assumed) before this is used in a live demo.
