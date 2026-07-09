# Windows Baseline Support — Design Spec

**Date:** 2026-07-09
**Author:** mlowcher61
**Status:** Approved (brainstorming)
**Target repo:** github.com/mlowcher61/baseline-validation
**Builds on:** [`2026-06-16-baseline-validation-design.md`](2026-06-16-baseline-validation-design.md)

## 1. Purpose

Extend the existing RHEL baseline validation solution to also validate Windows
hosts, without forking into a separate repository. Same product, same report
format, one repo — a customer with a mixed RHEL/Windows estate runs two job
templates against the same inventory and gets the same PASS/NON-COMPLIANT
report shape for either platform.

## 2. Scope

In scope:
- A Windows-equivalent check list: OS build, CPU cores, memory, C: drive
  capacity (system category), and Windows Firewall enabled, RDP NLA enforced,
  built-in Administrator account disabled (security category).
- OS-branching inside the existing `baseline_validation` role — one role, one
  evaluation/report engine, two platform-specific checks lists and two
  platform-specific fact-gathering task files.
- A second AAP job template ("Validate Baseline - Windows"), a second Machine
  credential, and a second inventory group (`windows`), run in parallel with
  the existing RHEL template/credential/group — each scoped via `limit` to its
  own group.
- `group_vars/windows.yml` for the WinRM/CredSSP connection settings.
- README updates documenting the two-platform model.

Out of scope:
- A unified single job template that targets both platforms in one run (the
  user chose two parallel templates with separate credentials instead).
- Remediation of discrepancies (validation only, same as RHEL).
- Non-Windows, non-RHEL platforms.
- Renamed-group-membership edge cases (e.g. a domain-joined "Administrators"
  group swap) beyond the built-in Administrator account check.

## 3. Target platform

- Built for **Ansible Automation Platform**, not ansible-core (unchanged).
- Certified/validated content only: adds `ansible.windows` (Red Hat certified)
  alongside the existing `ansible.builtin`, `ansible.platform`,
  `ansible.controller`. No community collections.

## 4. Architecture

One repo, one role. Two AAP job templates point at the same project/playbook:

| | RHEL (existing) | Windows (new) |
|---|---|---|
| Job template | Validate Baseline - RHEL | Validate Baseline - Windows |
| Inventory limit | `rhel` group | `windows` group |
| Credential | Baseline RHEL Machine Credential | Baseline Windows Machine Credential |
| Connection | SSH (Ansible default, no group_vars needed) | WinRM/CredSSP via `group_vars/windows.yml` |

Both templates can be launched in parallel. Because they're limited to
disjoint groups, they never contend for the same hosts.

`group_vars/windows.yml`:
```yaml
ansible_connection: winrm
ansible_winrm_transport: credssp
ansible_port: 5986
ansible_winrm_server_cert_validation: ignore   # tighten per environment
```

## 5. Role structure

```
roles/baseline_validation/
├── defaults/main.yml         # baseline_checks_rhel, baseline_checks_windows,
│                              # all threshold vars, bv_* and bv_win_* safe defaults
├── meta/main.yml
└── tasks/
    ├── main.yml               # branches on ansible_facts['os_family']
    ├── rhel.yml                # ansible.builtin.setup, root-fs derivation,
    │                           # gather_ssh_facts.yml (when security enabled)
    ├── gather_ssh_facts.yml    # unchanged
    ├── windows.yml              # ansible.windows.setup, C: drive derivation,
    │                            # gather_windows_security_facts.yml (when security enabled)
    ├── gather_windows_security_facts.yml   # new
    ├── checks.yml               # picks baseline_checks_rhel or _windows by
    │                            # os_family, same evaluation loop as before
    └── report.yml                # unchanged, platform-agnostic
```

`tasks/main.yml` orchestration becomes:
1. Include `rhel.yml` or `windows.yml` based on `ansible_facts['os_family']`.
2. Evaluate checks (`checks.yml`) — selects the matching list.
3. Render report and gate (`report.yml`) — unchanged.

## 6. Windows checks

System category (`baseline_checks_windows`):

| Check | Threshold var | Notes |
|---|---|---|
| OS build | `baseline_win_os_min_build` (default `"10.0.20348"`) | Windows Server 2022; uses `version_ge` like RHEL's OS check |
| CPU cores | `baseline_cpu_min_cores` (shared with RHEL) | |
| Memory | `baseline_mem_min_gb` (shared with RHEL) | |
| C: drive capacity | `baseline_root_fs_min_gb` (shared with RHEL) | derived from Windows mount/volume facts |

Security category, gated by the same `baseline_security_checks_enabled`
toggle used today:

| Check | Threshold var | Gathered via |
|---|---|---|
| Windows Firewall enabled (all profiles) | `baseline_win_firewall_expected` (default `"yes"`) | `Get-NetFirewallProfile` |
| RDP NLA enforced | `baseline_win_rdp_nla_expected` (default `"yes"`) | Registry: `HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp\UserAuthentication` |
| Built-in Administrator account disabled | `baseline_win_local_admin_disabled_expected` (default `"yes"`) | `Get-LocalUser` filtered by SID suffix `-500` (not by name, since the account can be renamed) |

`gather_windows_security_facts.yml` runs these via
`ansible.windows.win_shell` PowerShell, each returning `ConvertTo-Json`
output parsed with `from_json` (cleaner than the regex approach
`gather_ssh_facts.yml` uses on `sshd -T` output). No `become`/`runas` is
used — the WinRM account is assumed to already hold local admin rights on
the target. **This assumption should be validated against a real Windows
host in implementation**; if it doesn't hold, the security-facts tasks will
need `become: true` / `become_method: runas`.

**Safe-default requirement:** every new `bv_win_*` fact must have a default
in `defaults/main.yml` (e.g. `"unknown"` / `false`), the same fix applied in
commit `6f5fb6d` for `bv_ssh_*`. Ansible templates the entire checks list —
including entries whose gathering task was skipped — so these vars can never
be left undefined.

**Verification needed during implementation:** this design assumes
`ansible.windows.setup` populates `ansible_facts['memtotal_mb']`,
`ansible_facts['processor_vcpus']`, and `ansible_facts['distribution_version']`
with the same key names as `ansible.builtin.setup` does on RHEL, and that the
C: drive is discoverable via an analogous `mounts`/volume fact. These should
be confirmed against a real Windows host as the first implementation step;
if key names differ, `defaults/main.yml`'s `fact_expr` values for the
Windows system checks will need adjusting accordingly.

## 7. AAP config-as-code & credentials

- `configure_aap/vars/credentials.yml` gains a second Machine credential:
  "Baseline Windows Machine Credential" (org Demo). No secrets in git, same
  as the existing RHEL credential.
- `configure_aap/vars/inventories.yml` — no change to the inventory object
  itself; the `rhel` and `windows` groups and their host membership are
  created when hosts are added in AAP (not config-as-code, matching how the
  RHEL group's hosts are added today).
- `configure_aap/vars/job_templates.yml` gains a second job template
  ("Validate Baseline - Windows") with its own credential, `limit: windows`,
  and its own survey using the Windows threshold variables from §6 (shared
  vars like `baseline_cpu_min_cores` reuse the same survey question shape as
  the RHEL template; `baseline_security_checks_enabled` keeps the same
  variable name and toggles the Windows security checks the same way it
  today toggles the SSH ones).

## 8. Testing

- `ansible-lint` / `yamllint` clean, same as today.
- RHEL: unchanged local-testing story (`ansible_connection=local` against
  `localhost`).
- Windows: **no equivalent local-testing trick exists** — WinRM requires a
  real reachable Windows target. This is a documented limitation, not an
  automated test gap to close; the README will state that Windows checks
  must be exercised against a real or test Windows host reachable over
  WinRM/CredSSP.
- Config-as-code syntax-checked as today; applying it requires a live AAP
  instance.

## 9. Success criteria

- A customer with both RHEL and Windows hosts in one AAP inventory can run
  "Validate Baseline - RHEL" and "Validate Baseline - Windows" independently
  (including in parallel) and get the same PASS/NON-COMPLIANT report shape
  for either platform.
- Adding a new Windows check requires editing only `baseline_checks_windows`
  data, no task changes — same extensibility promise as the RHEL list.
- A new Ansible user can read the README and understand why there are two
  groups, two credentials, and two templates instead of one.
