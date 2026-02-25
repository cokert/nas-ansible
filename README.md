# nas-ansible

Ansible playbook to fully configure `cokert-B550M-ITX-ac` (ASRock B550M-ITX/ac, ZFS NAS).

Applying this to a fresh Ubuntu 24.04 install produces the same system as the current
running configuration. All source material audited in `~/nas-audit/`.

---

## Critical: Ubuntu installer ISO version

**Use Ubuntu 24.04.0 or 24.04.1** (ships with kernel 6.8).

Ubuntu 24.04.2+ ISOs default to the HWE kernel (6.11/6.14/6.17). ZFS 2.2.2 (Ubuntu's
packaged version) fails to build its DKMS module on kernel 6.9+ due to a block device API
change — pool will not be accessible. See [Launchpad #2117125](https://bugs.launchpad.net/bugs/2117125).

The playbook handles this automatically:
- Removes HWE kernel metapackages if installed
- Holds `linux-generic`, `linux-image-generic`, `linux-headers-generic` to prevent future HWE upgrades

If you only have a 24.04.2+ ISO: install normally, then run `sudo apt install linux-generic`
and reboot into 6.8 *before* running this playbook. The ZFS pool won't be importable until
you're on a supported kernel.

---

## Pre-rebuild checklist

Before wiping the OS SSD:

1. **Export the ZFS pool**
   ```bash
   sudo zpool export tank
   ```

2. **Remove the old Tailscale node** from the [Tailscale admin console](https://login.tailscale.com/admin/machines).
   The new machine must register as `nas` — if the old node is still present,
   Tailscale registers the new one as `nas-1` with no error, and Prometheus
   scraping breaks without alerting.

3. **Shut down / wipe / reinstall Ubuntu 24.04** (see ISO note above).

4. After first boot, run the playbook. The `tailscale` role joins automatically
   using the vault auth key and registers with `--hostname nas`.

---

## Playbook execution options

### Option A — Run on the NAS itself (simplest)

Good for: after wiping only the OS SSD, on the same machine.

```bash
# 1. Fresh Ubuntu 24.04 install (desktop-minimal recommended)
# 2. During install: create user 'cokert' (uid 1000)
# 3. After first boot:
sudo apt install ansible git
git clone <this-repo> ~/nas-ansible
cd ~/nas-ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml --ask-vault-pass
```

### Option B — Run from a laptop

Good for: remote management from your Mac.

Prereq: SSH key must be on the NAS (copy it manually or use `ssh-copy-id` with password auth
temporarily enabled).

```bash
# On your laptop:
brew install ansible   # or pip install ansible
git clone <this-repo> ~/nas-ansible
cd ~/nas-ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i inventory/remote.ini site.yml --ask-vault-pass
```

---

## Dry run / check mode

`--check` runs the playbook without making any changes and reports what would differ.
`--diff` shows the actual file content diff for any config that would be modified.

```bash
# Dry run with diffs (most useful)
ansible-playbook site.yml --check --diff --ask-vault-pass

# Dry run a single role
ansible-playbook site.yml --check --diff --tags samba --ask-vault-pass
```

**Caveat:** the ZFS role uses raw `command` tasks (`zpool`, `zfs`) which Ansible can't
simulate — those tasks are skipped or report misleading results in check mode. Skip the ZFS
role for a reliable dry run of everything else:

```bash
ansible-playbook site.yml --check --diff --skip-tags zfs --ask-vault-pass
```

Roles that work reliably in check mode: `system`, `users`, `ssh`, `samba`, `smartd`,
`smart_tests`, `nut`, `exporters`. The `packages` role is reliable for package state but
`apt upgrade` output is approximate.

---

## Secrets (Ansible Vault)

Sensitive values live in `group_vars/all/vault.yml`. Fill them in, then encrypt:

```bash
# Fill in real values first:
nano group_vars/all/vault.yml

# Encrypt (you'll be prompted for a vault password):
ansible-vault encrypt group_vars/all/vault.yml

# To edit later:
ansible-vault edit group_vars/all/vault.yml
```

Required secrets:
| Variable | What it is |
|----------|------------|
| `vault_samba_password_cokert` | Samba password for user cokert |
| `vault_samba_password_tm` | Samba password for user tm (TimeMachine) |
| `vault_nut_password` | NUT upsmon password (connect to homebridge:3493) |
| `vault_tailscale_auth_key` | Tailscale one-time auth key (for unattended join) |

The vault file is in `.gitignore` by default (unencrypted). Once encrypted, commit the
encrypted version.

---

## ZFS pool

### Normal case (fresh OS install, data drives untouched)

The playbook imports the existing pool by default:

```bash
ansible-playbook site.yml
# zfs role: zpool import -f tank  →  pool reappears on /tank
```

### If you need to create a brand-new pool (destructive!)

Only needed if the data drives are being reused from scratch. This will **erase all data**
on those drives.

```bash
ansible-playbook site.yml -e zfs_create_pool=true
```

Drive identifiers are in `group_vars/all/vars.yml` under `zfs_pool_drives`.

---

## Running individual roles

Each role has a tag matching its name:

```bash
ansible-playbook site.yml --tags packages     # just install packages
ansible-playbook site.yml --tags samba        # just reconfigure Samba
ansible-playbook site.yml --tags exporters    # redeploy docker compose stack
ansible-playbook site.yml --tags smart_tests  # redeploy SMART test timers
```

---

## Post-playbook manual steps

These aren't automated (require interactivity or one-time setup):

1. **SSH host key mismatch** — after a rebuild the NAS gets a new SSH host key,
   and clients will refuse to connect with a scary "REMOTE HOST IDENTIFICATION
   HAS CHANGED" warning. Fix on each client machine:
   ```bash
   ssh-keygen -R cokert-B550M-ITX-ac
   # or if connecting by Tailscale name:
   ssh-keygen -R nas
   ssh-keygen -R nas.jackalope-peacock.ts.net
   ```
   The next `ssh` will prompt to accept the new fingerprint as normal.

2. **Fan control** — requires interactive hardware detection:
   ```bash
   sudo sensors-detect
   sudo pwmconfig
   ```

3. **Samba passwords** — if samba users already existed and passwords need resetting:
   ```bash
   sudo smbpasswd cokert
   sudo smbpasswd tm
   ```

---

## Role overview

| Role | What it does |
|------|-------------|
| `system` | Sets hostname, /etc/hosts |
| `packages` | Adds Docker + Tailscale apt repos, installs all packages |
| `users` | Creates groups (media/tm/timemachine), user tm, sets cokert's groups, authorized_keys, sudoers |
| `ssh` | Deploys sshd_config (key-only auth) |
| `zfs` | Imports or creates pool, ensures datasets exist with correct quotas/properties |
| `samba` | Creates /srv dirs with permissions, deploys smb.conf, sets samba user passwords |
| `smartd` | Deploys smartd.conf, ensures smartd is running |
| `smart_tests` | Deploys 17 systemd units (template + 8 per-drive services + 8 timers) |
| `nut` | Deploys nut.conf and upsmon.conf (password from vault), starts nut-client |
| `exporters` | Creates /opt/exporters/, deploys docker-compose.yml + nginx.conf, runs `docker compose up -d` |

---

## Monitoring stack

The `exporters` role deploys:
- **nginx** (port 80) — reverse proxy for all metric endpoints
- **node-exporter** (port 9110, host network) — system metrics
- **smartctl-exporter** (port 9633) — SMART attributes + self-test metrics
- **zfs-exporter** (port 9134) — ZFS pool/dataset metrics

Prometheus scrapes the NAS via Tailscale MagicDNS. The Tailscale node name
is **`nas`**, set explicitly by the playbook via `tailscale up --hostname nas`
(independent of the system hostname). FQDN: `nas.jackalope-peacock.ts.net`.
The scrape config on the Prometheus host uses:

- `http://nas.jackalope-peacock.ts.net/node/metrics`
- `http://nas.jackalope-peacock.ts.net/smart/metrics`
- `http://nas.jackalope-peacock.ts.net/zfs/metrics`

**The Tailscale node name `nas` must be preserved across rebuilds** — if the
new machine registers under a different name, Prometheus scraping fails and
alerting fires. See the pre-rebuild checklist below.

Custom images: `ghcr.io/cokert/smartctl-exporter`, `ghcr.io/cokert/zfs-exporter`
(must be accessible from ghcr.io on the NAS at apply time).

---

## SMART self-tests

4x WD Red Pro 10TB drives, scheduled via systemd timers:

| Schedule | Drives |
|----------|--------|
| Daily 03:00-03:15 (staggered 5 min apart) | Short test, all 4 drives |
| Monday 03:00 | Long test, A2SJ (mirror-0) |
| Tuesday 03:00 | Long test, JATJ (mirror-1) |
| Wednesday 03:00 | Long test, ELRJ (mirror-0) |
| Thursday 03:00 | Long test, JS3J (mirror-1) |

Samsung SSD excluded from self-tests (passive SMART monitoring only via smartd DEVICESCAN).

---

## Plan (implementation reference)

### Execution model choice

| | Laptop control machine | NAS runs itself (localhost) |
|---|---|---|
| SSH prereq on fresh install | Yes — need key or password first | No |
| Ansible install location | Laptop | NAS: `sudo apt install ansible` |
| Expand to multiple machines | Easy | Needs inventory restructure |
| Recommended for single NAS | | Yes |

Both modes supported: `inventory/hosts.ini` (localhost) and `inventory/remote.ini` (over SSH).

### Design decisions

- **Roles over flat playbook** — each concern is independently runnable via `--tags`
- **ZFS pool guard** — defaults to `zpool import`; requires explicit `-e zfs_create_pool=true` for destructive create
- **Samba user idempotency** — checks `pdbedit -L` before calling `smbpasswd -a` to avoid duplicate adds
- **NUT as Jinja2 template** — password injected from vault at deploy time
- **No textfile collector** — docker exporters (smartctl, zfs) replace all textfile scripts
- **restart: always in compose** — Docker restarts containers on daemon restart; no wrapper systemd unit needed
