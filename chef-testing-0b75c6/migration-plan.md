# MIGRATION FROM CHEF TO ANSIBLE

**Executive Summary**

This repository is a small‑scale Chef code base that provisions a multi‑site Nginx reverse‑proxy, a caching layer (Memcached & Redis), and a FastAPI tutorial application backed by PostgreSQL. The stack targets both Ubuntu 18.04+ and CentOS 7+ and is exercised via Vagrant (Fedora 42) with Chef Solo for local development. Migration to Ansible will replace Chef cookbooks with Ansible roles/playbooks, retain the same functional outcomes, and simplify provisioning for both on‑premises and cloud environments.

**Scope** – 3 Chef cookbooks, 1 Berksfile, Vagrant configuration, and supporting scripts.
**Estimated Timeline** – 3 weeks (1 week discovery & design, 1 week implementation, 1 week testing & hand‑over).
**Team** – 1 senior Ansible engineer (lead), 1 devops engineer (testing), 1 security analyst (credential review).

---

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

| Module | Description | Path | Technology | Key Features |
|--------|-------------|------|------------|--------------|
| **cache** | Configures caching services – installs Memcached, configures Redis (including authentication), creates log directories and applies a hacky config cleanup for Redis. | `cookbooks/cache` | Chef | Memcached install, Redis with password, directory management, inline Ruby block for config sanitisation. |
| **fastapi-tutorial** | Deploys a FastAPI tutorial application: installs system packages, clones source, creates a Python virtual environment, installs dependencies, provisions PostgreSQL, creates DB/user, writes `.env`, creates a systemd service, enables and starts the app. | `cookbooks/fastapi-tutorial` | Chef | Python/venv, Git checkout, PostgreSQL service & DB creation, hard‑coded DB credentials, systemd unit file, service enable/start. |
| **nginx-multisite** | Sets up an Nginx reverse‑proxy serving multiple SSL‑enabled sub‑domains. Includes security hardening (fail2ban, ufw, sysctl), Nginx config/templates, per‑site document roots, self‑signed certificate generation, and site content provisioning. | `cookbooks/nginx-multisite` | Chef | Nginx install, fail2ban, ufw firewall rules, sysctl security, per‑site config, SSL cert generation, templated nginx.conf, site HTML files. |

**CRITICAL PATH VERIFICATION:** All paths have been confirmed from the repository tree and content reads; no additional modules were hidden.

### Infrastructure Files

- `Berksfile` – Declares cookbook dependencies (`nginx`, `memcached`, `redisio`). Will be replaced by Ansible Galaxy/requirements.yml.
- `Vagrantfile` – Provides VM definition (Fedora 42 box, networks, libvirt resources) and invokes `vagrant-provision.sh`. Will be kept for local dev but the provisioning step will switch from Chef Solo to an Ansible playbook.
- `vagrant-provision.sh` – Bash script that installs Chef, runs Berkshelf, and executes `chef-solo`. Will be refactored to install Ansible (via `pip` or system package) and run the generated Ansible playbook.
- `solo.rb` & `solo.json` – Chef Solo configuration files (not shown but referenced). These will be replaced by Ansible inventory and variable files.

---

## Target Details

- **Operating System**: Ubuntu 18.04+ and CentOS 7+ are explicitly supported in each cookbook's `metadata.rb`. The Vagrant box is Fedora 42; the migration will target a generic Linux family (Debian/RedHat) using Ansible `ansible_os_family` facts.
- **Virtual Machine Technology**: Vagrant with libvirt provider (Fedora 42 box). No cloud‑specific resources are present.
- **Cloud Platform**: Not specified. The playbooks will be cloud‑agnostic, but optional variables can be added for AWS, Azure, etc., in the future.

---

## Migration Approach

### Key Dependencies to Address

| Dependency | Current Source | Ansible Equivalent |
|------------|----------------|--------------------|
| `nginx` cookbook | `Berksfile` (nginx ~> 12.0) | `geerlingguy.nginx` role (or custom role) |
| `memcached` cookbook | `Berksfile` (memcached ~> 6.0) | `geerlingguy.memcached` role |
| `redisio` cookbook | `Berksfile` (redisio ~> 7.2.4) | `geerlingguy.redis` role |
| `fail2ban` (via package) | Manual package install in security.rb | `geerlingguy.fail2ban` role |
| `ufw` firewall | Manual `execute` resources | `ansible.posix.ufw` module |
| Systemd services | `service` resources | `systemd` module |
| Git checkout | `git` resource | `git` module |
| Python virtualenv | `execute` resource | `ansible.builtin.pip` + `virtualenv` module |

External cookbook code will be removed and replaced by mature Ansible Galaxy roles where possible, falling back to custom roles for logic not covered by existing roles (e.g., the Redis config hack, per‑site SSL cert generation).

### Security Considerations

| Area | Observation | Migration Recommendation |
|------|-------------|---------------------------|
| **Hard‑coded credentials** | `cache/recipes/default.rb` contains Redis password `redis_secure_password_123`. `fastapi-tutorial/recipes/default.rb` contains DB user/password `fastapi` / `fastapi_password`. | Move secrets to Ansible Vault or an external secrets manager (e.g., HashiCorp Vault). Replace inline values with `{{ vault_redis_password }}` and `{{ vault_fastapi_db_password }}`. |
| **Self‑signed certificates** | `nginx-multisite/recipes/ssl.rb` generates self‑signed certificates for each site. | Keep generation logic but externalize the domain list to variables; optionally replace with real certificates via Ansible `acme_certificate` module for Let’s Encrypt. |
| **Firewall rules** | `ufw` commands are executed directly. | Use Ansible `ufw` module with idempotent rule definitions. |
| **Package installation** | Packages installed via Chef `package` resource. | Use Ansible `package` module; define OS‑specific package lists via `vars` and `when` clauses. |
| **File permissions** | Various `mode`, `owner`, `group` settings. | Preserve using Ansible `file` module with explicit `mode`, `owner`, `group`. |

### Technical Challenges

1. **Redis configuration hack** – The original cookbook patches the Redis config file with a Ruby block. In Ansible we must replicate this idempotently, likely using the `lineinfile` or `blockinfile` module with `create`/`replace` logic. Testing will be required to ensure no unintended line removals.
2. **Per‑site SSL cert generation** – The Ruby `execute` resource uses a heredoc; Ansible will need a `command` or `openssl_certificate` custom task. Ensuring idempotence (no regeneration if files exist) is essential.
3. **Variable mapping** – Chef attributes (`node['nginx']['sites']`, `node['nginx']['ssl']`, etc.) are stored in JSON/attributes files. We must translate these to Ansible `group_vars`/`host_vars` (YAML) and ensure naming consistency.
4. **Vagrant provisioning switch** – The existing `vagrant-provision.sh` script assumes Chef. We'll need to replace it with Ansible installation steps and `ansible-playbook` execution, while keeping the same Vagrantfile structure.

### Migration Order

1. **Application layer** – Migrate `fastapi-tutorial` first; it depends on PostgreSQL (system packages) and the caching services.
2. **Foundational services** – Migrate `cache` cookbook next (memcached & Redis). This is low‑risk, isolated, and provides a base for the FastAPI DB connections.
3. **Ingress & security** – Finally migrate `nginx-multisite`, as it ties together site content, SSL, firewall, and fail2ban. This order respects runtime dependencies and allows incremental testing.

---

## Assumptions

- The repository does not contain hidden Puppet, Salt, or PowerShell modules; all infrastructure is defined via Chef.
- No external secret management system is currently used; passwords are hard‑coded and will need to be vaulted.
- The Vagrant box (Fedora 42) is only used for local development; production environments will use either Ubuntu or CentOS as indicated by the `supports` statements.
- All required external cookbooks are available on Chef Supermarket and have comparable Ansible Galaxy roles.
- No additional environment‑specific data bags, roles, or encrypted items exist beyond what is visible in the provided files.
- The `solo.rb` and `solo.json` files contain standard Chef Solo configuration and do not embed extra logic.

---

## Next Steps

1. **Create Ansible role skeletons** (`cache`, `fastapi_tutorial`, `nginx_multisite`).
2. **Export cookbook attributes to Ansible variables** and store them under `group_vars/all.yml`.
3. **Implement tasks** mirroring each Chef recipe, leveraging existing Galaxy roles where possible.
4. **Migrate secrets to Ansible Vault** and update playbooks to reference vault variables.
5. **Refactor Vagrant provisioning** to install Ansible and run the new playbook.
6. **Run integration tests** inside the Vagrant VM, validate functional parity.
7. **Document migration** and hand over to the operations team.

---

*Prepared by the Migration Planning Agent.*