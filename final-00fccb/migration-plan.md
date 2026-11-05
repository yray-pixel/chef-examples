# MIGRATION FROM CHEF TO ANSIBLE

**Executive Summary**
This repository is a Chef codebase that provisions a multi‑site Nginx reverse‑proxy, caching services (Memcached & Redis), and a FastAPI tutorial application with PostgreSQL. The repository consists of three cookbooks, a Berksfile that pins external dependencies, and Vagrant configuration for local development. Migration to Ansible will involve translating each cookbook into one or more Ansible roles, consolidating shared variables into group/host vars, and replacing Chef‑specific resources (e.g., `template`, `service`, `execute`) with Ansible modules. The overall effort is moderate – three core cookbooks (≈ 200 lines of Ruby code each) plus supporting Vagrant/VM configuration. Estimated timeline: **4‑6 weeks** (1 week discovery & planning, 2 weeks role implementation, 1 week testing & CI integration, 1 week documentation & hand‑over).

---

## Module Migration Plan

This repository contains **Chef** cookbooks that each require an individual migration path.

### MODULE INVENTORY

| Module | Description | Path | Technology | Key Features |
|--------|-------------|------|------------|--------------|
| **cache** | Configures caching services – installs Memcached, configures Redis with authentication, creates Redis log directory, and applies a bespoke Ruby block to clean up Redis config. | `cookbooks/cache` | Chef | Memcached integration, Redis setup with password, custom config sanitisation, log directory management |
| **fastapi-tutorial** | Deploys a FastAPI Python application: installs system packages, clones repo, creates Python virtual‑env, installs requirements, sets up PostgreSQL service & DB/user, writes `.env` with DB credentials, creates a systemd service unit for the app. | `cookbooks/fastapi-tutorial` | Chef | Python/virtualenv provisioning, Git checkout, PostgreSQL user/database creation, application systemd service, inline secrets (DB password) |
| **nginx-multisite** | Sets up Nginx as a multi‑site reverse proxy with SSL, Fail2Ban, UFW firewall hardening, sysctl security tweaks, and per‑site document roots. | `cookbooks/nginx-multisite` | Chef | Nginx package & templated configuration, multi‑domain site provisioning, Fail2Ban, UFW rules, SSH hardening, sysctl security settings |

**CRITICAL PATH VERIFICATION:**
All modules above were discovered via the repository tree and verified by reading each cookbook’s entrypoint (`recipes/default.rb`) and metadata.

---

### Infrastructure Files

| File | Purpose & Migration Considerations |
|------|------------------------------------|
| `Berksfile` | Declares external cookbook dependencies (`nginx`, `memcached`, `redisio`). In Ansible these map to Galaxy roles or community collections (e.g., `geerlingguy.nginx`, `geerlingguy.redis`). Dependency version constraints must be captured in `requirements.yml`. |
| `Vagrantfile` | Provides a local development VM (Fedora 42) with libvirt provider, private network, port forwarding, and a provisioning script. When moving to Ansible, the Vagrantfile can be retained for local testing, but the Ansible playbook should be callable via `ansible-playbook -i inventory.yml`. |
| `vagrant-provision.sh` | Shell script executed by Vagrant to install Chef; will be replaced by an Ansible provisioning step (installing required packages, then running the new playbook). |
| `solo.json` | Chef‑solo run‑list and attribute overrides (sites, SSL paths, security flags). These become Ansible inventory variables (`nginx_sites`, `nginx_ssl`, `security_fail2ban_enabled`, etc.). |
| `solo.rb` | Chef‑solo configuration (cookbook path, log level). Not required for Ansible; analogous settings belong in `ansible.cfg`. |

---

## Target Details

- **Operating System**: The cookbooks support Ubuntu ≥ 18.04 and CentOS ≥ 7. The Vagrant box is Fedora 42. For migration we recommend targeting **Ubuntu 22.04 LTS** as the primary OS (most community Ansible roles assume Debian‑based platforms). CentOS compatibility can be preserved via conditional tasks.
- **Virtual Machine Technology**: Development utilizes **Vagrant with libvirt** provider. Production may run on any cloud or on‑prem VM; the migration plan does not bind to a specific hypervisor.
- **Cloud Platform**: Not explicitly referenced. No cloud‑specific resources detected (e.g., AWS CLI, Azure modules). The playbook will remain cloud‑agnostic.

---

## Migration Approach

### Key Dependencies to Address

| Dependency | Current Source | Ansible Replacement |
|------------|----------------|---------------------|
| `nginx` cookbook | `cookbooks/nginx` (external) | `geerlingguy.nginx` collection role |
| `memcached` cookbook | `memcached` (external) | `geerlingguy.memcached` role |
| `redisio` cookbook | `redisio` (external) | `geerlingguy.redis` role |
| Ruby helpers (`ruby_block`) | Inline Ruby script for Redis config sanitisation | Ansible `lineinfile` / `replace` modules, or a custom Jinja2 template to generate a clean `redis.conf` |

### Security Considerations

1. **Hard‑coded credentials**
   - Redis password (`redis_secure_password_123`) – present in `cookbooks/cache/recipes/default.rb`.
   - FastAPI PostgreSQL user/password (`fastapi` / `fastapi_password`) – defined in the `execute` block inside `cookbooks/fastapi-tutorial/recipes/default.rb` and written to `/opt/fastapi-tutorial/.env`.
   - **Migration approach**: Store such secrets in Ansible Vault (`ansible-vault create group_vars/all/vault.yml`) and reference them via `{{ vault_redis_password }}` etc. Remove hard‑coded values from playbooks.
2. **SSH hardening** – Chef modifies `/etc/ssh/sshd_config` based on attributes (`disable_root`, `password_auth`). In Ansible this is handled by the `ansible.posix.authorized_key` module for keys and `lineinfile` for sshd settings, ensuring idempotency.
3. **UFW & Fail2Ban** – Both are enabled/configured via Chef; translate to `community.general.ufw` and `community.general.fail2ban` roles, ensuring rule ordering and idempotent execution.
4. **SSL certificate directories** – References to `/etc/ssl/certs` and `/etc/ssl/private`. Verify that certificates are provisioned securely, possibly using Ansible `copy` with `mode: 0600` or integrating with a PKI solution like HashiCorp Vault.

### Technical Challenges

- **Idempotent translation of custom Ruby blocks** – The `ruby_block` that strips several lines from Redis config must be rewritten using Ansible’s `replace` or a Jinja2 template. Testing will be needed to ensure functional parity.
- **Variable mapping** – Chef attributes (`node['nginx']['sites']`, `node['security']['ssh']`) are supplied via `solo.json`. These need to be converted into structured Ansible variables (`nginx_sites`, `security_ssh_disable_root`, etc.) and placed in appropriate inventory/group vars.
- **Systemd service creation** – The FastAPI service file is generated via a `file` resource with embedded heredoc. In Ansible this will use the `template` module, which simplifies future updates.
- **File ownership & permissions** – Chef explicitly sets owners/modes for many resources. Ensure Ansible tasks replicate these (`owner`, `group`, `mode`). Some tasks (e.g., creating the Redis log dir) may be collapsed into a single role responsibility.
- **Testing across OS families** – While the cookbooks claim Ubuntu and CentOS support, Ansible role conditionals must be added (`when: ansible_os_family == 'Debian'` etc.) and tested on both families.

### Migration Order

1. **Foundation (Week 1‑2)** – Create a base Ansible playbook skeleton with inventory, `ansible.cfg`, and common role structure. Migrate external dependencies first by pulling corresponding Galaxy roles (`geerlingguy.nginx`, `geerlingguy.redis`, `geerlingguy.memcached`). Verify they install correctly on the target OS.
2. **Cache Role (Week 2‑3)** – Convert `cookbooks/cache` into an `cache` role. Implement Redis password via Vault, ensure log directory creation, and replace the Ruby block with `replace` tasks.
3. **Nginx‑Multisite Role (Week 3‑4)** – Translate `nginx-multisite` into an `nginx_multisite` role. Map site definitions from `solo.json` to `nginx_sites` variable, use Jinja2 templates for `nginx.conf`, `site.conf`, and security files. Add UFW and Fail2Ban tasks using community modules.
4. **FastAPI Role (Week 4‑5)** – Convert `fastapi-tutorial` into a `fastapi` role. Include Python package installation, virtualenv creation (`ansible.builtin.python_virtualenv`), git checkout, systemd service templating, and PostgreSQL DB/user creation via the `community.postgresql.postgresql_user` and `postgresql_db` modules. Secure database credentials with Vault.
5. **Integration & Testing (Week 5‑6)** – Assemble a master playbook that calls the three roles in dependency order (`cache` → `nginx_multisite` → `fastapi`). Run the playbook against the Vagrant VM, iteratively fixing idempotency issues. Add CI pipeline (GitHub Actions) to lint (`ansible-lint`) and test the playbook.

### Assumptions

- The target production environment will be a **Linux (Ubuntu 22.04)** host with `systemd`.
- No Windows or PowerShell components exist in this repo; only Chef‑Ruby code is present.
- All external cookbooks in the Berksfile have functional equivalents in Ansible Galaxy; if not, custom roles will be written.
- Secrets currently hard‑coded will be moved to **Ansible Vault**; the team has Vault workflow in place.
- The Vagrant development VM is only a test harness; production will be provisioned directly with Ansible (no Chef client required).
- No cloud‑specific resources are present; any future cloud integration will be added later.

---

## Summary

By converting each Chef cookbook into a dedicated Ansible role, consolidating variables into inventory, and securing credentials with Ansible Vault, the organization can achieve a clean, idempotent, and maintainable automation stack. The proposed 4‑6 week timeline balances development effort with thorough testing, providing a solid foundation for future expansion (e.g., adding monitoring, scaling, or cloud provisioning).