# MIGRATION FROM CHEF TO ANSIBLE

**Executive Summary**
The repository is a small‑to‑medium Chef codebase that provisions a development‑grade web stack consisting of:
- Caching layer (Memcached + Redis) – `cookbooks/cache`
- A FastAPI Python application with PostgreSQL – `cookbooks/fastapi-tutorial`
- A multi‑site Nginx reverse‑proxy with security hardening, firewall rules, and self‑signed SSL – `cookbooks/nginx-multisite`

The migration will replace these Chef cookbooks with Ansible roles/playbooks while preserving the same functional outcome. The codebase is modest (three cookbooks, ~300 lines of Ruby), but it touches several subsystems (system packages, services, firewall, TLS, systemd, database init, and secret handling). A realistic effort is 2 – 3 weeks for a single engineer, plus a brief overlap for testing and knowledge transfer.

---

## Module Migration Plan

This repository contains **Chef** cookbooks that need individual migration planning:

### MODULE INVENTORY

| Module | Description | Path | Technology | Key Features |
|--------|-------------|------|------------|--------------|
| **cache** | Configures caching services – installs and configures Memcached and Redis, creates Redis log directory, applies a custom hack to strip unwanted directives from the Redis config, and enables Redis service. | `cookbooks/cache` | Chef | Memcached inclusion, Redis with password (`redis_secure_password_123`), directory management, config‑file post‑processing |
| **fastapi-tutorial** | Deploys a FastAPI tutorial application: installs system packages, clones a Git repo, creates a Python virtual‑env, installs requirements, provisions a PostgreSQL database/user, writes an `.env` file with DB credentials, creates a systemd unit, and ensures the service runs. | `cookbooks/fastapi-tutorial` | Chef | Python 3, virtualenv, PostgreSQL DB & user (`fastapi`/`fastapi_password`), systemd service, environment file |
| **nginx-multisite** | Sets up an Nginx reverse‑proxy capable of serving multiple SSL‑enabled sub‑domains. Installs Nginx, renders core `nginx.conf`, creates per‑site document roots, copies static index files, installs Fail2Ban & UFW firewall with default‑deny policy, hardens SSH, generates self‑signed certs for each site, and applies sysctl security tweaks. | `cookbooks/nginx-multisite` | Chef | Multi‑site Nginx config, Fail2Ban, UFW rules, SSH hardening, self‑signed cert generation, sysctl security profile |

**CRITICAL PATH VERIFICATION:**
All modules and paths have been verified directly from the repository tree and the primary entrypoint files (`default.rb`, `nginx.rb`, `security.rb`, `ssl.rb`). No external or hidden modules were inferred.

---

### Infrastructure Files

| File | Purpose | Migration Considerations |
|------|---------|--------------------------|
| `Berksfile` | Lists cookbook dependencies (nginx, memcached, redisio) and local cookbook paths. | Convert to Ansible Galaxy/requirements.yml. External cookbooks become Ansible collections or custom roles. |
| `Vagrantfile` | Defines a Vagrant VM for local test/development. | Map to Ansible‑compatible Vagrant provisioning (still usable) or to cloud‑init/VMware/other IaC. |
| `solo.rb` / `solo.json` | Chef Solo configuration for node attributes and run‑list. | Replace with Ansible inventory variables and playbook execution command. |
| `vagrant-provision.sh` | Shell script executed by Vagrant to install Chef and run the cookbooks. | Replace with an Ansible provisioner call (`ansible-playbook`) or embed provisioning steps in the Vagrantfile using the Ansible provisioner.

---

## Target Details

- **Operating System**: The cookbooks declare support for Ubuntu ≥ 18.04 and CentOS ≥ 7. The Vagrantfile uses a generic base box (not shown but typically Ubuntu 20.04). For the migration we will target **Ubuntu 22.04 LTS** (common, recent, and fully supported by Ansible). If the production environment differs, the role variables can be overridden per OS.
- **Virtual Machine Technology**: Vagrant is used for local testing. No explicit cloud provider is referenced, so the target platform is “On‑premise / Vagrant / Generic VM”. The migration plan will keep Vagrant for CI/testing, using Ansible’s built‑in Vagrant provisioner.
- **Cloud Platform**: Not specified. No AWS/Azure/GCP SDK usage detected.

---

## Migration Approach

### Key Dependencies to Address

| Dependency | Version / Constraint | Ansible Replacement |
|------------|----------------------|---------------------|
| `nginx` | `~> 12.0` (Chef Supermarket) | Ansible `geerlingguy.nginx` collection or custom role mirroring the current templates (`nginx.conf`, site config). |
| `memcached` | `~> 6.0` | Ansible `geerlingguy.memcached` role. |
| `redisio` | `~> 7.2.4` | Ansible `geerlingguy.redis` role, with extra `redis.conf` tweaks via a template or lineinfile. |
| `fail2ban` (installed via package) | N/A | Ansible `geerlingguy.fail2ban` role. |
| `ufw` (firewall) | N/A | Ansible `community.general.ufw` module or a custom role to apply the same rule set. |

### Security Considerations

- **Hard‑coded credentials**: 
  - Redis password (`redis_secure_password_123`) is embedded in `cache/recipes/default.rb`.
  - PostgreSQL user/password (`fastapi`/`fastapi_password`) are defined in the fastapi cookbook.
  - These values should be moved to Ansible Vault or sourced from an external secret manager (e.g., HashiCorp Vault, AWS Secrets Manager). The migration plan includes creating encrypted variables (`vault_redis_password`, `vault_fastapi_db_password`).
- **Self‑signed certificates**: Generated on‑the‑fly by the `ssl.rb` recipe. In Ansible we can keep the same generation logic using the `openssl_certificate` module or a shell command, but production should replace them with real certificates managed by a PKI.
- **SSH hardening**: The security recipe toggles `PermitRootLogin` and `PasswordAuthentication` based on node attributes (`node['security']['ssh']`). These attributes will become Ansible variables (`ssh_disable_root`, `ssh_password_auth`). Ensure they are applied early in the playbook to avoid lock‑out.
- **Firewall defaults**: The `ufw` rules are idempotent but rely on command‑line checks (`not_if`). Ansible’s `ufw` module is idempotent and will replace those guards.
- **Systemd unit**: The FastAPI service file is written using the `file` resource. In Ansible we will use the `template` module with a Jinja2 template and the `systemd` module to enable/start the service.

### Technical Challenges

1. **Chef attribute system vs. Ansible variables** – Chef uses a deep attribute hierarchy (`node['nginx']['sites']`, `node['security']['ssh']['disable_root']`). Translating this to Ansible requires building a variable structure (e.g., `nginx_sites:` dictionary). Care must be taken to preserve default values and allow overrides.
2. **Ruby `ruby_block` hack for Redis config** – This manipulates the Redis config file via Ruby string replacement. In Ansible we can achieve the same with the `replace` or `lineinfile` module, but we need to ensure order of operations (create file, then patch). Testing will be required to guarantee functional parity.
3. **Dynamic site generation** – The Nginx cookbook iterates over a node attribute list to create document roots and copy per‑site static `index.html` files. Ansible will need a loop with the `with_dict` construct and a template or `copy` module. Static files must be packaged inside the role’s `files/` directory, mirroring the existing structure.
4. **Idempotence of package installation** – Chef’s `package` resource is idempotent; Ansible’s `apt`/`yum` modules are also idempotent but require the correct package manager per OS. We'll abstract this via `ansible_pkg_mgr` fact.
5. **Testing in Vagrant** – Existing Vagrant workflow runs Chef Solo. We'll transition the Vagrantfile to use the Ansible provisioner, ensuring the same VM image and network configuration.

### Migration Order

1. **Foundation Layer (infra)** – Migrate `Berksfile` dependencies to Ansible Galaxy requirements, set up inventory, and validate the Vagrant‑Ansible provisioning pipeline.
2. **Cache Cookbook** – Convert `cache` to an Ansible role (`role_cache`). This is low‑risk (packages and services) but includes the Redis config hack, providing a good learning step.
3. **Nginx‑Multisite** – Migrate the Nginx role next, as it builds on the firewall and SSH hardening already covered. The role will include templates for `nginx.conf`, `security.conf`, and per‑site configs, plus self‑signed cert generation.
4. **FastAPI‑Tutorial** – Finally, migrate the application role, which depends on the database service and systemd unit creation. This role also introduces secret handling, so moving to Ansible Vault will be validated at this stage.

---

## Assumptions

- The target environment will remain a Linux VM (Ubuntu 22.04) with `systemd` as init system.
- No additional external services (e.g., load balancers, external DB clusters) are required beyond what is defined in the cookbooks.
- All attributes used by the Chef cookbooks are defined within the cookbooks themselves or default node attributes; there are no hidden data bags or environments.
- The self‑signed certificates generated by the Nginx role are acceptable for the development / testing environment. Production will replace them with proper certs.
- The Vagrant base box used locally matches the OS version we will target with Ansible.
- No Windows hosts are present; therefore PowerShell, DSC, or other Windows‑specific technologies are out of scope.
- Secrets currently hard‑coded will be migrated to Ansible Vault; the team has Vault set up and appropriate access controls.

---

*Prepared by the Migration Planning Agent – 2026‑10‑28*
