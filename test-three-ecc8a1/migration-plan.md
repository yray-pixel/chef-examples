# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository is a **Chef Solo**-based infrastructure-as-code project that provisions a single Vagrant VM (Fedora 42 / libvirt) running three local cookbooks and two external Supermarket dependencies. The run-list deploys an Nginx multi-site reverse proxy with SSL, a dual caching layer (Memcached + Redis), and a FastAPI application backed by PostgreSQL.

The migration scope is **moderate**: three local cookbooks, two external cookbook dependencies, one Vagrant-driven test environment, and a handful of security hardening concerns. No Chef Server, Policyfiles, encrypted data bags, or Test Kitchen harness are present — the entire stack is driven by `chef-solo` with a flat `solo.json` attribute file, which maps cleanly to Ansible variables and playbooks.

**Estimated migration effort**: 2–3 weeks for a single engineer, or 1 week with a two-person team.

---

## Module Migration Plan

This repository contains **3 local Chef cookbooks** that need individual migration planning, plus **2 external Supermarket cookbooks** that must be replaced with Ansible equivalents.

---

### MODULE INVENTORY

**CRITICAL PATH VERIFICATION:**
All paths below were confirmed from the provided repository tree and file reads. No paths have been inferred or invented.

---

- **nginx-multisite**
  - **Description**: Nginx web server provisioning for three SSL-enabled virtual hosts (`test.cluster.local`, `ci.cluster.local`, `status.cluster.local`). Manages the full Nginx lifecycle: package installation, global `nginx.conf` from an ERB template, per-site `sites-available`/`sites-enabled` symlink management, self-signed TLS certificate generation via `openssl`, security hardening (fail2ban, UFW firewall rules, sysctl kernel parameters, SSH hardening), and static site content deployment.
  - **Path**: `cookbooks/nginx-multisite`
  - **Technology**: Chef
  - **Key Features**:
    - ERB templates: `nginx.conf.erb`, `site.conf.erb`, `security.conf.erb`, `sysctl-security.conf.erb`, `fail2ban.jail.local.erb`
    - Self-signed RSA-2048 certificate generation per vhost (development only)
    - UFW firewall: default-deny, allow SSH/HTTP/HTTPS
    - fail2ban installation and configuration
    - SSH hardening: `PermitRootLogin no`, `PasswordAuthentication no`
    - sysctl security tuning via `/etc/sysctl.d/99-security.conf`
    - Static `index.html` files deployed per site from `files/default/{ci,status,test}/`
    - Custom `lineinfile` LWRP resource (`resources/lineinfile.rb`)
    - Attribute-driven site map (document roots, SSL toggle) via `attributes/default.rb`
    - Depends on external `nginx ~> 12.0` Supermarket cookbook

- **cache**
  - **Description**: Dual caching layer that installs and configures both Memcached and Redis on the same host. Redis is configured with password authentication (`requirepass`) on port 6379. Includes a `ruby_block` workaround that post-processes the generated Redis config file to strip deprecated `replica-*` directives incompatible with the installed Redis version.
  - **Path**: `cookbooks/cache`
  - **Technology**: Chef
  - **Key Features**:
    - Delegates Memcached setup to the `memcached ~> 6.0` Supermarket cookbook
    - Delegates Redis setup to the `redisio ~> 7.2.4` Supermarket cookbook
    - Hardcoded Redis password: `redis_secure_password_123` (plaintext in recipe)
    - Post-install config-file mutation hack to remove deprecated Redis directives
    - Redis log directory creation (`/var/log/redis`)
    - Depends on external `memcached` and `redisio` Supermarket cookbooks

- **fastapi-tutorial**
  - **Description**: Full-stack Python application provisioner. Clones a FastAPI application from GitHub, creates a Python 3 virtual environment, installs pip dependencies, configures a PostgreSQL database with a dedicated user, writes a `.env` file with database credentials, and registers a systemd service (`fastapi-tutorial.service`) that starts the app via `uvicorn` on port 8000.
  - **Path**: `cookbooks/fastapi-tutorial`
  - **Technology**: Chef
  - **Key Features**:
    - System packages: `python3`, `python3-pip`, `python3-venv`, `git`, `postgresql`, `postgresql-contrib`, `libpq-dev`
    - Git clone from `https://github.com/dibanez/fastapi_tutorial.git` (branch: `main`)
    - Python venv at `/opt/fastapi-tutorial/venv`
    - PostgreSQL user `fastapi` with hardcoded password `fastapi_password` (plaintext in recipe)
    - Hardcoded `DATABASE_URL` written to `/opt/fastapi-tutorial/.env` (mode `0644` — world-readable)
    - systemd unit file written inline; service runs as `root` (security concern)
    - No idempotency guard on `create_db_user` execute block (uses `|| true` workaround)

---

### Infrastructure Files

- `Berksfile`: Berkshelf dependency manifest. Declares all three local cookbooks by path and pins external dependencies (`nginx ~> 12.0`, `memcached ~> 6.0`, `redisio ~> 7.2.4`). Replace with Ansible Galaxy `requirements.yml` or inline role tasks.
- `solo.json`: Chef Solo node attribute file and run-list definition. This is the primary source of truth for site configuration (vhost names, document roots, SSL paths, security flags). Migrate to Ansible `group_vars/all.yml` or a dedicated `vars/` file.
- `solo.rb`: Chef Solo configuration (cache path, cookbook path, log level). No Ansible equivalent needed — superseded by `ansible.cfg` and inventory.
- `Vagrantfile`: Vagrant VM definition using `generic/fedora42` box with libvirt provider (2 vCPU, 2 GB RAM). Provisions via shell script. Migrate to an Ansible-compatible `Vagrantfile` using the `ansible` provisioner, or replace entirely with a local inventory pointing at the VM.
- `vagrant-provision.sh`: Bootstrap shell script that installs Chef, runs Berkshelf to vendor dependencies, then executes `chef-solo`. Replace with a direct `ansible-playbook` invocation or Vagrant's built-in Ansible provisioner.

---

### Target Details

- **Operating System**: Ubuntu ≥ 18.04 and CentOS ≥ 7.0 are declared as supported platforms in all three `metadata.rb` files. The Vagrant test environment uses `generic/fedora42` (Fedora 42). The `www-data` user and `ufw`/`apt-get` references in recipes indicate a primary Ubuntu/Debian target. Default to **Ubuntu 22.04 LTS** for Ansible role development; add RHEL/Fedora conditionals where package names diverge.
- **Virtual Machine Technology**: **libvirt / KVM** — explicitly configured in the `Vagrantfile` via `config.vm.provider "libvirt"`. Vagrant is used solely as a local development/test harness; it is not part of the production provisioning model.
- **Cloud Platform**: Not specified. No cloud-provider SDKs, metadata endpoint references, or cloud-init configurations are present in the repository.

---

## Migration Approach

### Key Dependencies to Address

- **nginx (Supermarket ~> 12.0)**: The `nginx-multisite` cookbook delegates package installation and service management to this community cookbook. Replace with the `ansible.builtin.package` module (`nginx`) and `ansible.builtin.service`. The ERB templates (`nginx.conf.erb`, `site.conf.erb`, `security.conf.erb`) translate directly to Ansible Jinja2 templates (`.j2`). Consider the community role `geerlingguy.nginx` as a drop-in if opinionated Nginx management is preferred.

- **memcached (Supermarket ~> 6.0)**: Wraps Memcached installation and service management. Replace with `ansible.builtin.package` + `ansible.builtin.service`. The `geerlingguy.memcached` Ansible Galaxy role is a direct equivalent.

- **redisio (Supermarket ~> 7.2.4)**: Manages Redis installation, config file generation, and service enablement. The post-install config-mutation hack in `cache::default` exists because `redisio` generates deprecated directives for the installed Redis version — this workaround should be eliminated entirely in Ansible by writing a clean `redis.conf` Jinja2 template directly. Consider `geerlingguy.redis` or manage Redis natively with `ansible.builtin.template` + `ansible.builtin.service`.

- **Berkshelf / `berks vendor`**: The dependency resolution and vendoring workflow has no direct Ansible equivalent. Replace with `ansible-galaxy install -r requirements.yml` for any adopted community roles, or inline all logic into local roles.

---

### Security Considerations

- **Hardcoded Redis password** (`cache::default`): The string `redis_secure_password_123` is committed in plaintext in `cookbooks/cache/recipes/default.rb`. Migrate to **Ansible Vault** (`ansible-vault encrypt_string`) and reference via a vaulted variable (e.g., `vault_redis_password`).

- **Hardcoded PostgreSQL credentials** (`fastapi-tutorial::default`): The PostgreSQL user password `fastapi_password` and the full `DATABASE_URL` (including credentials) are written in plaintext in the recipe and deployed to `/opt/fastapi-tutorial/.env` with mode `0644` (world-readable). In Ansible: store credentials in Vault, render the `.env` file via a Jinja2 template, and tighten file permissions to `0600` owned by the application user.

- **Self-signed TLS certificates**: All three vhosts use `openssl req -x509` to generate self-signed certificates at provision time. This is explicitly noted as development-only. For production Ansible roles, integrate **Let's Encrypt / Certbot** (`community.crypto.acme_certificate`) or a PKI-backed certificate distribution mechanism. The certificate and key paths (`/etc/ssl/certs`, `/etc/ssl/private`) are parameterised in `solo.json` and attributes — these map cleanly to Ansible variables.

- **SSH hardening** (`nginx-multisite::security`): `PermitRootLogin no` and `PasswordAuthentication no` are applied via `sed` on `/etc/ssh/sshd_config`. Migrate to `ansible.builtin.lineinfile` or the `devsec.hardening.ssh_hardening` Galaxy role for idempotent, auditable SSH configuration.

- **UFW firewall** (`nginx-multisite::security`): Default-deny with explicit allow for ports 22, 80, 443. Migrate to `community.general.ufw` module. Ensure the Ansible control node's SSH port is allowed before enabling default-deny to avoid lockout.

- **sysctl kernel hardening** (`nginx-multisite::security`): Security parameters written to `/etc/sysctl.d/99-security.conf` via ERB template. Migrate to `ansible.posix.sysctl` module or a Jinja2 template with `ansible.builtin.template` + `ansible.builtin.command: sysctl -p`.

- **FastAPI service runs as root** (`fastapi-tutorial::default`): The systemd unit sets `User=root`. This is a significant security risk. The Ansible migration should create a dedicated unprivileged system user (e.g., `fastapi`) and run the service under that account.

- **`.env` file world-readable** (`fastapi-tutorial::default`): Mode `0644` exposes `DATABASE_URL` (with password) to all local users. Fix to `0600` in the Ansible template task.

---

### Technical Challenges

- **`redisio` config-mutation hack**: The `ruby_block "fix_redis_config"` in `cache::default` post-processes the Redis config file to strip directives that `redisio` generates but the installed Redis version rejects. This is a brittle workaround. In Ansible, bypass this entirely by owning the `redis.conf` Jinja2 template directly — no external cookbook, no post-processing needed.

- **Custom `lineinfile` LWRP** (`nginx-multisite/resources/lineinfile.rb`): The cookbook ships its own `lineinfile` resource, likely because the Chef community equivalent was unavailable or insufficient. Ansible's `ansible.builtin.lineinfile` module is a native, first-class primitive — this resource has a direct, idiomatic replacement.

- **Vagrant + libvirt test environment**: The current test loop is `vagrant up` → shell bootstrap → `chef-solo`. Migrating the test harness to use Vagrant's `ansible` provisioner (or Molecule with a libvirt/vagrant driver) is a prerequisite for validating the migrated roles before production use.

- **Fedora 42 vs. Ubuntu/CentOS target mismatch**: The Vagrantfile uses `generic/fedora42` but the cookbooks target Ubuntu ≥ 18.04 / CentOS ≥ 7. Package names (`ufw` is Ubuntu-specific; Fedora uses `firewalld`), service names, and user names (`www-data` vs. `nginx`) differ. Ansible roles must include OS-family conditionals (`ansible_os_family`, `ansible_distribution`) or separate variable files per distro.

- **PostgreSQL idempotency**: The `create_db_user` execute block uses `|| true` to suppress errors on re-runs. Ansible's `community.postgresql.postgresql_user` and `community.postgresql.postgresql_db` modules are natively idempotent and should replace this pattern entirely.

- **Git-based application deployment**: The `fastapi-tutorial` cookbook clones from a public GitHub URL at provision time. In a production Ansible workflow, consider pinning to a specific commit SHA or tag (not `main`) and evaluating whether a CI/CD artifact deployment model is more appropriate than live Git clones.

- **No secrets management infrastructure**: There are no Chef Vault, encrypted data bags, or external secrets backends in use. All secrets are plaintext in recipes or `solo.json`. The Ansible migration must introduce **Ansible Vault** from day one — there is no existing secrets workflow to preserve or migrate.

---

### Migration Order

The following order minimises risk by starting with the most self-contained cookbook and ending with the one carrying the most security debt:

1. **`nginx-multisite`** *(Priority 1 — highest value, self-contained)*
   Translate the five ERB templates to Jinja2, replace `execute`-based UFW/sysctl/SSH hardening with idiomatic Ansible modules, and wire up the attribute-driven vhost loop as an Ansible `loop` over a `nginx_sites` variable list. This cookbook delivers the most visible infrastructure and has no hardcoded secrets.

2. **`cache`** *(Priority 2 — moderate complexity, one security fix required)*
   Replace `memcached` and `redisio` Supermarket cookbook delegation with direct Ansible tasks or Galaxy roles. Eliminate the `ruby_block` config-mutation hack by owning the Redis config template. Move the Redis password to Ansible Vault before the role is committed.

3. **`fastapi-tutorial`** *(Priority 3 — highest security debt, most remediation needed)*
   Migrate Python/venv/git/systemd tasks to Ansible modules (`ansible.builtin.pip`, `ansible.builtin.git`, `ansible.builtin.systemd`). Replace `community.postgresql` execute blocks with proper modules. Create a dedicated `fastapi` system user. Move all credentials to Ansible Vault. Fix `.env` file permissions. Pin the Git revision to a specific commit or tag.

---

### Assumptions

1. **Target OS for Ansible roles**: Ubuntu 22.04 LTS is assumed as the primary target based on `www-data` user references and `ufw`/`apt-get` usage, despite the Vagrantfile using Fedora 42. Clarification is needed on whether Fedora/RHEL support is a hard requirement or only used for local Vagrant testing.

2. **No Chef Server**: The repository uses `chef-solo` exclusively. It is assumed there is no Chef Server, Chef Automate, or Hosted Chef organisation to decommission as part of this migration.

3. **Development-only SSL**: Self-signed certificates are treated as development scaffolding only. It is assumed that production environments will require a proper PKI or ACME-based certificate solution, which is out of scope for the initial Ansible role but must be addressed before production deployment.

4. **Single-node topology**: `solo.json` and the Vagrantfile describe a single VM running all services (Nginx, Memcached, Redis, PostgreSQL, FastAPI). It is assumed this topology is intentional for the current scope. A multi-host inventory design is not required unless explicitly requested.

5. **Secrets management starting point**: No existing Vault, HashiCorp Vault, AWS Secrets Manager, or equivalent is in use. Ansible Vault (file-based) is assumed as the initial secrets solution. If an external secrets backend is available in the target environment, the plan should be updated accordingly.

6. **`redisio` version compatibility hack scope**: The `ruby_block` that strips deprecated Redis directives suggests the installed Redis version is newer than what `redisio ~> 7.2.4` was designed for. The exact Redis version on the target OS is unknown; the Ansible template should be validated against the actual Redis version available in the target OS package repositories.

7. **FastAPI application ownership**: The `fastapi-tutorial` cookbook clones from a public third-party GitHub repository (`dibanez/fastapi_tutorial`). It is assumed this repository is under the team's control or that the migration team has authority to pin or fork it. If it is a genuine external dependency, a mirroring or artifact strategy should be defined.

8. **Vagrant as dev/test harness only**: The Vagrantfile and `vagrant-provision.sh` are assumed to be local developer tooling, not part of any CI pipeline or production provisioning path. The Ansible migration will target the same VM for local testing but via `ansible-playbook` directly or Molecule.

9. **`nginx ~> 12.0` Supermarket cookbook role**: The `nginx-multisite` cookbook depends on the external `nginx` cookbook but the local recipes also install the `nginx` package directly. It is assumed the external cookbook is used primarily for its service resource and that its functionality can be fully replaced by Ansible's `package` and `service` modules without loss of functionality.

10. **No existing Ansible infrastructure**: It is assumed the team has no pre-existing Ansible control node, inventory, or Galaxy namespace. The migration will establish these from scratch.
