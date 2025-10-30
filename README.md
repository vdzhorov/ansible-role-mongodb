# Ansible Role MongoDB

Install and configure **MongoDB 8.0.15 (LTS)** on **Ubuntu 24.04 (Noble)** with authentication, user management and production-grade OS tuning.

> ✅ Version is **pinned** to `8.0.15` by default (overridable).  
> ✅ Idempotent user management via `mongosh`.

---

## Table of Contents

- [Features](#features)
- [Compatibility](#compatibility)
- [Role Variables](#role-variables)
- [Quick Start](#quick-start)
- [Security & Credentials](#security--credentials)
- [TLS / Encryption](#tls--encryption)
- [Version Pinning & Upgrades](#version-pinning--upgrades)
- [Operational Tasks](#operational-tasks)
- [Troubleshooting](#troubleshooting)
- [Tags](#tags)
- [License](#license)

---

## Features

- Installs **MongoDB 8.0.15** from the official MongoDB APT repository.
- Enables **SCRAM** authentication and **keyfile** for internal member auth.
- Creates/updates **admin** and **application users**.
- **OS tuning** for production (nofile/nproc limits, THP disabled, low swappiness).
- Clean separation of **defaults**, **vars**, **tasks**, **handlers**, **templates**, and **meta** (with `argument_specs.yml`).

---

## Compatibility

- **OS:** Ubuntu **24.04 (Noble)**
- **Ansible:** 2.15+ (see `meta/main.yml`)
- **Architectures:** `amd64`, `arm64` (as supported by MongoDB packages)

---

## Role Variables

> Full, typed variable list lives in `meta/argument_specs.yml`. Below are the most commonly tuned vars (with defaults):

```yaml
mongodb_version: "8.0.15"
mongodb_repo_series: "8.0"           # Repository channel
mongodb_service_name: mongod
mongodb_bind_ip: "0.0.0.0"
mongodb_port: 27017
mongodb_dbpath: "/var/lib/mongodb"
mongodb_logpath: "/var/log/mongodb/mongod.log"

# Auth
mongodb_enable_auth: true
mongodb_keyfile_path: "/etc/mongod-keyfile"
mongodb_keyfile_content: "{{ lookup('password', inventory_dir ~ '/.mongodb_keyfile chars=ascii_letters,digits length=48') }}"

# Admin + app users
mongodb_admin_user: "admin"
mongodb_admin_pwd: "ChangeMe_SuperSecret"  # Vault this
mongodb_users: []                     # See examples below

# OS tuning
mongodb_nofile_limit: 64000
mongodb_nproc_limit: 32000
mongodb_swappiness: 1
mongodb_disable_thp: true

# TLS (optional)
mongodb_tls_enabled: false
mongodb_tls_mode: "requireTLS"        # preferTLS|requireTLS|disabled
mongodb_tls_cert_file: "/etc/ssl/mongodb/server.pem"
mongodb_tls_ca_file: "/etc/ssl/mongodb/ca.pem"
```

## Quick Start

Single Node Inventory:

```ini
[mongo_single]
mongo1 ansible_host=10.0.0.21
```

Group vars `group_vars/mongo_single.yml`:

```yaml
mongodb_replset_enabled: false           # single node
mongodb_bind_ip: "0.0.0.0"
mongodb_admin_pwd: "{{ vault_mongodb_admin_pwd }}"
mongodb_users:
  - name: appuser
    pwd: "{{ vault_appuser_pwd }}"
    db: appdb
    roles:
      - { role: readWrite, db: appdb }

```

Playbook:

```yaml
- name: Provision single-node MongoDB
  hosts: mongo_single
  become: true
  roles:
    - mongodb_ubuntu24
```

## Security & Credentials

- Vault everything sensitive: admin password, app user passwords, keyfile. Example:

    ```bash
    ansible-vault create group_vars/mongo_rs/vault.yml
    ```

    ```yaml
    # group_vars/mongo_rs/vault.yml
    vault_mongodb_admin_pwd: "S3cureAdmin!"
    vault_app_rw_pwd: "S3cureAppRW!"
    vault_app_ro_pwd: "S3cureAppRO!"
    vault_mongodb_keyfile: "UvmH5y...same-on-all-members..."
    ```

- Ensure firewall rules expose TCP 27017 only to trusted peers/app tiers.
- Consider enabling TLS (see next section).

## TLS / Encryption

Enable TLS by toggling:

```yaml
mongodb_tls_enabled: true
mongodb_tls_mode: requireTLS          # or preferTLS
mongodb_tls_cert_file: /etc/ssl/mongodb/server.pem
mongodb_tls_ca_file: /etc/ssl/mongodb/ca.pem
```

Notes:

- `server.pem` should contain private key + certificate chain.
- Ensure file permissions are restricted to `mongodb` and readable by the service.
- Clients must connect with `--tls` / URI `tls=true` and trust the CA.

## Version Pinning & Upgrades

- The role pins packages to {{ mongodb_version }} and applies apt-mark hold.
- To upgrade later:
    1. Plan the rolling upgrade (read MongoDB’s release notes for 8.0.x).
    2. Update your var: `mongodb_version: "8.0.16"` (example).
    3. Temporarily unhold, upgrade, then hold again:

        ```yaml
        # Option A: run the role after updating mongodb_version (role manages pinning)
        # Option B: manual snippet if needed:
        - name: Unhold MongoDB packages
        ansible.builtin.dpkg_selections:
            name: "{{ item }}"
            selection: install
        loop:
            - mongodb-org
            - mongodb-org-database
            - mongodb-org-server
            - mongodb-org-mongos
            - mongodb-org-tools
        ```

    4. Run the role; it will install the new pinned version and re-hold packages.
- Always upgrade secondaries first, then step down primary and upgrade it.

## Operational Tasks

### Add a New Application User

Update `mongodb_users` and re-run the role:

```yaml
mongodb_users:
  - name: reporting
    pwd: "{{ vault_reporting_pwd }}"
    db: appdb
    roles:
      - { role: read, db: appdb }
```

### Rotate Admin Password

Change `mongodb_admin_pwd` and re-run. The task calls `updateUser`, so it’s idempotent.

### Rotate Keyfile (internal auth)

1. Generate new value for mongodb_keyfile_content (identical on all members).
2. Apply to all nodes with a rolling restart:
    - Apply new key to secondaries, restart each.
    - Step down primary, apply new key, restart, allow to re-elect.

### Backup & Restore (example commands)

```bash
# Backup
mongodump --host mongo1 --port 27017 -u admin -p '...' --authenticationDatabase admin -o /backups/$(date +%F)

# Restore
mongorestore /backups/2025-10-30
```

## Troubleshooting

mongod won’t start after config change

- Check `/var/log/mongodb/mongod.log`.
- Validate `/etc/mongod.conf` syntax (YAML). The role’s template avoids duplicate keys.

Users not created/updated

- Verify `mongodb_admin_user/mongodb_admin_pwd` are correct and vaulted.
- Check Ansible output for `created:` or `updated:` lines per user.

TLS handshake problems

- Confirm clients trust the CA and are using `--tls` / `tls=true` in connection strings.
- Ensure certificateKeyFile includes private key + cert chain.

## Tags

This role does not define Ansible tags by default. You can include/import specific task files in your own wrapper playbooks to simulate tagging (e.g., only `users.yml`).

## License

MIT
