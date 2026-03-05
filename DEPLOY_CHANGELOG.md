# BeeGFS Deploy Changelog and Issues

Date: 2026-03-05
Scope: `/root/code/deploy-beegfs`

## What Was Changed

1. Repository and package installation
- Installed `ansible-core` on control-node.
- Installed collection `ansible.posix` required by `mount` task.
- Fixed BeeGFS apt repo URL format in role (`repo.yml`): removed invalid `/dists/<codename>` from base URL.
- Switched default BeeGFS version to `8`.
- Added `beegfs-tools` package to install list so `beegfs` CLI exists.

2. BeeGFS 8 compatibility fixes
- `setup_mgmtd.yml` updated to support both:
  - BeeGFS 7: `beegfs-setup-mgmtd`
  - BeeGFS 8: `beegfs-mgmtd --init`
- Added BeeGFS 8 mgmtd toml settings:
  - `auth-file = /etc/beegfs/conn.auth`
  - `auth-disable = true|false` (from var)
  - `tls-disable = true` (lab mode)
  - `license-disable = true` (lab mode)

3. Management host resolution fixes
- `beegfs_mgmt_host` default changed to prefer `ansible_host` of mgmt node.
- Enforced `sysMgmtdHost = {{ beegfs_mgmt_host }}` in:
  - `beegfs-client.conf`
  - `beegfs-meta.conf`
  - `beegfs-storage.conf`
- This removes dependency on DNS name `nodeA`.

4. Authentication workflow fixes
- Rewrote `roles/beegfs/tasks/auth.yml`:
  - Removed broken `item is file` checks.
  - Added dynamic detection of existing config files via `stat`.
  - Uses `lineinfile` for `connDisableAuthentication` / `connAuthFile`.
  - For auth-enabled mode: generate on mgmt node, `slurp`, distribute with `copy`, mode `0600`.
- Unified auth file path to `/etc/beegfs/conn.auth` in defaults.

5. Verification playbook fixes (`verify.yml`)
- Added CLI resolution task (`beegfs` binary discovery).
- Health commands run with `--tls-disable`.
- Adjusted steps that return `rc=1` in license-disabled mode but still print valid output:
  - `health capacity`
  - `node list --with-nics`
  - now `failed_when: false` + assert non-empty output.
- Replaced obsolete BeeGFS v7 check (`beegfs-ctl --listnodes`) with BeeGFS 8 `beegfs node list` parsing.

## Errors Encountered and Resolution

1. `ansible-playbook: command not found`
- Cause: Ansible not installed on control-node.
- Fix: `dnf install -y ansible-core`.

2. `couldn't resolve module/action 'mount'`
- Cause: missing collection.
- Fix: `ansible-galaxy collection install ansible.posix`.

3. BeeGFS GPG URL/repo errors (`404`, missing Release)
- Cause: incorrect version/url composition.
- Fix: moved to BeeGFS 8 repo and corrected apt repo URL format.

4. `beegfs-setup-mgmtd` missing
- Cause: BeeGFS 8 changed initialization model.
- Fix: conditional init with `beegfs-mgmtd --init`.

5. `beegfs-mgmtd` failed due to missing auth/tls files
- Cause: defaults expect auth file and TLS certs.
- Fix: manage `conn.auth` path and disable TLS/licensing in lab mode.

6. `beegfs-storage` / `beegfs-client` failed on nodeB (hostname `nodeA` unresolved)
- Cause: config contained hostname not resolvable.
- Fix: enforce `sysMgmtdHost` with mgmt IP from inventory.

7. Verify failures with BeeGFS 8 CLI and license-disabled mode
- Cause: old commands (`beegfs-ctl`) and strict RC handling.
- Fix: migrate checks to `beegfs` CLI and validate output instead of RC only.

## Current Status

- `ansible-playbook -i inventory/hosts.ini playbook.yml`: PASS
- `ansible-playbook -i inventory/hosts.ini verify.yml`: PASS
- Services active on both nodes:
  - `beegfs-mgmtd` (nodeA)
  - `beegfs-meta` (nodeA)
  - `beegfs-storage` (nodeA,nodeB)
  - `beegfs-client` (nodeA,nodeB)
- BeeGFS mount present on both nodes (`/mnt/beegfs`).

## Notes / Known Lab Assumptions

- Role is intentionally apt/deb-focused.
- Health CLI uses `--tls-disable` because lab setup runs BeeGFS 8 mgmt without TLS certs.
- `license-disable = true` is enabled in mgmtd for lab simplicity.
