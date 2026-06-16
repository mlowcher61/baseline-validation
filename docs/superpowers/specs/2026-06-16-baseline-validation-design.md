# Baseline Validation — Design Spec

**Date:** 2026-06-16
**Author:** mlowcher61
**Status:** Approved (brainstorming)
**Target repo:** github.com/mlowcher61/baseline-validation

## 1. Purpose

A reusable Ansible Automation Platform (AAP) solution that validates whether a
RHEL host meets a defined technical baseline (CPU, OS version, memory, storage)
and produces a clear **PASS / NON-COMPLIANT report that calls out each
discrepancy** in the AAP job output.

It serves as a proof-of-concept to justify a broader compliance-automation
engagement with a customer. A demo operator runs it from the AAP controller as a
job template with a survey, points it at a RHEL host, and the job output shows
either "all checks pass" or exactly which parameters fail the validated baseline.

## 2. Scope

In scope:
- Validate four baseline parameters against live `ansible_facts`:
  - OS version (minimum RHEL major.minor)
  - CPU core count (minimum)
  - Memory (minimum, GB)
  - Root filesystem capacity (minimum, GB)
- A data-driven check engine so additional checks are added as data, not logic.
- A human-readable report in the job output that calls out each discrepancy.
- Job pass/fail gating on compliance, with a toggle.
- Config-as-code to stand up the AAP objects needed to run the demo.
- README that fully explains the solution for a new Ansible user.

Out of scope (explicitly, for this POC):
- HTML/JSON report artifacts (terminal/job-output report only).
- Remediation of discrepancies (validation only).
- Multi-distro support beyond RHEL.
- Per-host baseline overrides beyond what the survey provides.

## 3. Target platform

- Built for **Ansible Automation Platform**, not ansible-core.
- Demoed entirely from the AAP controller: a job template + survey.
- Certified/validated content only: `ansible.builtin`, `ansible.platform`,
  `ansible.controller`.

## 4. Validation approach — data-driven checklist

The baseline is defined as a **list of check definitions** in
`roles/baseline_validation/defaults/main.yml` (config-as-code). Each check is a
mapping:

```yaml
baseline_checks:
  - name: "OS version"
    fact: "{{ ansible_facts['distribution_version'] }}"
    operator: "version_ge"      # version-aware >=
    threshold: "9.0"
    unit: ""
  - name: "CPU cores"
    fact: "{{ ansible_facts['processor_vcpus'] }}"
    operator: "ge"
    threshold: 2
    unit: "cores"
  - name: "Memory"
    fact: "{{ (ansible_facts['memtotal_mb'] / 1024) | round(1) }}"
    operator: "ge"
    threshold: 4
    unit: "GB"
  - name: "Root filesystem capacity"
    fact: "{{ root_fs_size_gb }}"   # derived from ansible_facts['mounts']
    operator: "ge"
    threshold: 40
    unit: "GB"
```

Supported operators: `ge`, `le`, `eq` (numeric) and `version_ge`, `version_eq`
(version-aware, using the `version` test). A single task loops the list,
evaluates `fact <operator> threshold`, and records a per-check result
(`name`, `found`, `threshold`, `unit`, `pass`).

**Trade-off accepted:** more abstract than four explicit tasks, but the customer
adds checks by editing data with no new logic. That extensibility is the upsell
and keeps the role clean as the baseline grows.

Default baseline (all overridable via the AAP survey):
**OS ≥ RHEL 9.0, CPU cores ≥ 2, memory ≥ 4 GB, root filesystem ≥ 40 GB.**

## 5. Behavior & output

1. Gather facts (`ansible.builtin.setup`).
2. Derive helper facts (e.g., root filesystem size in GB from `mounts`).
3. Evaluate **every** check — never stop at the first failure.
4. Print one report block per host via `ansible.builtin.debug`:

   ```
   BASELINE VALIDATION — web01.example.com
   =========================================
   [ PASS ] CPU cores              : 4    (min 2 cores)
   [ FAIL ] OS version             : 8.6  (min 9.0)   <-- does not meet validated baseline
   [ PASS ] Memory                 : 8.0  (min 4 GB)
   [ PASS ] Root filesystem capacity: 80  (min 40 GB)
   -----------------------------------------
   RESULT: NON-COMPLIANT  (1 of 4 checks failed)
   ```

5. After printing the report, if the host is non-compliant the play **fails the
   host** so AAP shows a red/failed job (automated gating — the demo moment).
   Compliant hosts pass green.

A single toggle var, `baseline_fail_on_noncompliance` (default `true`), flips
between fail-the-job and report-only without editing tasks. This can be exposed
on the survey so the behavior can be changed live.

## 6. Repository structure

```
baseline-validation/
├── README.md                       # full explanation for a new Ansible user
├── ansible.cfg
├── collections/requirements.yml    # ansible.platform, ansible.controller, ansible.builtin
├── playbooks/validate_baseline.yml
├── roles/baseline_validation/
│   ├── defaults/main.yml           # baseline definition (config-as-code) + toggle
│   ├── meta/main.yml
│   └── tasks/
│       ├── main.yml                # orchestration: facts -> derive -> evaluate -> report -> gate
│       ├── checks.yml              # data-driven evaluation loop
│       └── report.yml              # render + print the report, gate on compliance
└── configure_aap/                  # config-as-code to stand up AAP for the demo
    ├── README.md
    ├── configure.yml
    └── vars/
        ├── organizations.yml
        ├── projects.yml
        ├── inventories.yml
        ├── credentials.yml         # machine credential definition (no secrets in git)
        └── job_templates.yml       # job template + survey spec
```

## 7. AAP config-as-code & credentials

- `configure_aap/` uses **`ansible.platform`** for resources it covers
  (organization, execution environment, project) and **`ansible.controller`**
  only for controller-specific resources (inventory, job template, survey),
  honoring the "ansible.platform when possible" rule.
- The target RHEL host is reached via a standard **Machine credential attached
  to the job template** — no vaulted secrets committed to git. The config-as-code
  defines the credential object; the secret values are supplied in AAP at apply
  time (e.g., via controller env vars / extra prompts), never stored in the repo.
- Baseline thresholds and the fail toggle are exposed through a **survey** so a
  demo operator can dial them in live and instantly turn a host
  compliant/non-compliant.

## 8. Testing

- `ansible-lint` clean against the repo profile.
- `yamllint` clean.
- A smoke run of `playbooks/validate_baseline.yml` against `localhost`
  (the RHEL-like control node) proving: the report renders for a compliant host,
  a deliberately raised threshold flips the host to NON-COMPLIANT and calls out
  the discrepancy, and the `baseline_fail_on_noncompliance` toggle controls the
  job result.
- Config-as-code is validated for syntax (`ansible-playbook --syntax-check`);
  applying it requires a live AAP instance and is part of the demo runbook, not
  automated tests.

## 9. Success criteria

- Running the job template against a RHEL host produces a clear report that
  either confirms all checks pass or names each failing parameter as not meeting
  the validated baseline.
- A new Ansible user can read the README and understand how to configure AAP and
  run the demo.
- Adding a new baseline check requires editing only the `baseline_checks` data.
