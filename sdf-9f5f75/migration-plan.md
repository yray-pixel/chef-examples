# MIGRATION FROM CHEF TO ANSIBLE

**Executive Summary**

The repository is a Chef-based provisioning setup that installs and configures a multi‑site Nginx reverse‑proxy, a caching layer (Memcached + Redis), and a FastAPI tutorial application backed by PostgreSQL. All infrastructure is defined as three top‑level Chef cookbooks, wrapped in a Vagrant + Chef‑Solo environment. Migration to Ansible will involve translating each cookbook into an Ansible role, recreating the run‑list logic, and mapping external Chef‑Supermarket dependencies to Ansible Galaxy or native modules. The effort is moderate: three distinct functional areas, a handful of external cookbook dependencies, and a small amount of hard‑coded secrets that must be externalised. An estimated timeline is **3 weeks** (1 week for discovery & design, 1 week for implementation & testing, 1 week for validation & documentation).

---

## Module Migration Plan

This repository contains **Chef cookbooks** that need to be migrated individually into Ansible roles.

### MODULE INVENTORY

| Cookbook | Description | Path | Technology | Key Features |
|----------|-------------|------|------------|--------------|
| **cache** | Configures caching services – installs Memcached, sets up Redis with authentication, creates log directory, and applies a custom Ruby‑block hack to clean up Redis configuration defaults. | `cookbooks/cache` | Chef | Memcached installation, Redis server with password, custom configuration clean‑up script |
| **fastapi-tutorial** | Deploys a FastAPI tutorial application. Installs Python, clones source repo, creates a virtual environment, installs dependencies, provisions PostgreSQL, creates DB/user, writes `.env`, creates a systemd service for the app and enables it. | `cookbooks/fastapi-tutorial` | Chef | Python 3 stack, PostgreSQL DB creation, systemd service, environment file with DB credentials |
| **nginx-multisite** | Sets up Nginx with multiple SSL‑enabled sub‑domains, hardens the host (fail2ban, UFW, SSH config), generates self‑signed certificates for development, and deploys per‑site document roots and static index pages. | `cookbooks/nginx-multisite` | Chef | Nginx package & templated config, multi‑site configuration via node attributes, Fail2Ban, UFW firewall, SSH hardening, automated self‑signed SSL cert generation |

**CRITICAL PATH VERIFICATION:** All three cookbooks were verified from the repository tree and their primary recipe files (`default.rb` or equivalent) were examined.

---

### Infrastructure Files

| File | Purpose & Migration Considerations |
|------|--------------------------------------|
| `Berksfile` | Lists local cookbooks and external Supermarket dependencies (`nginx`, `memcached`, `redisio`). In Ansible, these map to Galaxy roles (`geerlingguy.nginx`, `geerlingguy.memcached`, `geerlingguy.redis`). Ensure version compatibility. |
| `solo.rb` | Chef‑Solo configuration (cookbook paths, log level). No direct Ansible equivalent; the Ansible inventory will replace this. |
| `solo.json` | Run‑list and attribute data that drive the Chef run. This becomes the Ansible playbook variables (e.g., `nginx_sites`, `security` dict). |
| `Vagrantfile` | Defines the development VM (Fedora 42 box, libvirt provider, network, synced folder, provisioning script). Ansible can be used as the provisioner in Vagrant or the VM can be built via a separate playbook. |
| `vagrant-provision.sh` | Shell script that installs Chef, Berkshelf, and runs Chef‑Solo. Replace with an Ansible provisioner block (`config.vm.provision "ansible"`) or invoke `ansible-playbook` from the script. |

---

## Target Details

- **Operating System**: The Vagrant box is `generic/fedora42` (Fedora 42). All cookbooks support Ubuntu ≥ 18.04 and CentOS ≥ 7, but the primary test environment is Fedora. Ansible playbooks should target the generic Linux family and include conditional package/module handling for Debian/RedHat families.
- **Virtual Machine Technology**: Libvirt provider is used (`config.vm.provider "libvirt"`). No specific cloud provider is referenced.
- **Cloud Platform**: Not specified. The playbooks will be written to be provider‑agnostic (VM‑based, on‑prem, or cloud).

---

## Migration Approach

### Key Dependencies to Address
- **External Cookbooks** → Ansible Galaxy roles
  - `nginx` → `geerlingguy.nginx` (or custom role if deep templating required)
  - `memcached` → `geerlingguy.memcached`
  - `redisio` → `geerlingguy.redis`
- **System Packages** (python3, postgresql, git, etc.) → `ansible.builtin.package` with appropriate `state: present`.
- **Git checkout** → `ansible.builtin.git`.
- **Systemd service** → `ansible.builtin.systemd`.
- **Firewall** → `community.general.ufw` module.
- **Fail2Ban** → `community.general.fail2ban` (or template + service).
- **SSL generation** → `community.crypto.openssl_certificate` or a custom command module.

### Security Considerations
- **Hard‑coded passwords**:
  - Redis password (`redis_secure_password_123`) in `cache/recipes/default.rb`.
  - FastAPI DB user/password (`fastapi_password`) in the `.env` file generated by `fastapi-tutorial`.
  - **Migration strategy**: Store these secrets in Ansible Vault or a centralized secret manager (e.g., HashiCorp Vault) and reference them via `{{ vault_redis_password }}` etc.
- **SSH hardening** – modifies `/etc/ssh/sshd_config`. Ansible can use `lineinfile` to enforce the same settings, but ensure idempotency.
- **Self‑signed certificates** – generated on‑the‑fly. In production the certificates should be replaced with proper CA‑signed certs; Ansible variables should allow toggling between dev and prod.
- **File permissions** – ensure Ansible tasks set correct ownership (`root:ssl-cert`, mode `0640`).

### Technical Challenges
1. **Chef attribute hierarchy vs. Ansible variables** – The `solo.json` file contains nested attribute structures (`nginx.sites`, `security.ssh`). Translating this into Ansible group/host vars and role defaults requires careful mapping.
2. **Ruby‑block custom Redis config hack** – Direct file manipulation using Ruby code. In Ansible this will be replaced with `replace` or `blockinfile` tasks; validation needed to ensure the same lines are removed.
3. **Idempotency of self‑signed certificate generation** – The Chef recipe uses `not_if` to avoid regeneration. In Ansible, a `creates` check or `stat` module will be required.
4. **Vagrant provisioner switch** – Existing shell script installs Chef. Re‑writing to use Ansible provisioner must handle the same bootstrap steps (package manager updates, required Python packages, etc.).

### Migration Order
1. **Infrastructure & Bootstrap** – Convert `Vagrantfile` and `vagrant-provision.sh` to use Ansible provisioner; set up inventory and basic playbook scaffolding.
2. **Cache role** – Migrate `cache` cookbook (external dependencies `memcached` and `redisio`). This role is largely independent and provides a good low‑risk starter.
3. **Nginx‑multisite role** – Migrate Nginx installation, templating, firewall, fail2ban, and SSL generation. It depends on the cache role for Redis (if any) but otherwise can be built next.
4. **FastAPI‑tutorial role** – Migrate the application stack, PostgreSQL setup, DB/user creation, and systemd service. This role depends on the cache role for Redis (optional) and on the Nginx role for site exposure, so it is the final piece.

### Assumptions
- The target environment will remain a VM (Fedora 42) for testing; production OS may differ but the Ansible roles will include OS‑family conditional logic.
- No external secret management system is currently in place; we assume the team will adopt Ansible Vault for storing passwords.
- All external cookbook functionality can be covered by existing Ansible Galaxy roles; custom logic (e.g., Redis config cleanup) will be re‑implemented with Ansible modules.
- The current `solo.json` attribute data is the single source of truth for site definitions and security settings.
- No additional hidden directories or cookbooks exist beyond those listed.

---

## Next Steps
1. **Create Ansible role skeletons** (`cache`, `nginx_multisite`, `fastapi_tutorial`).
2. **Map Chef attributes to Ansible variables** and create default `vars/main.yml` files.
3. **Replace Chef resources with Ansible modules** as outlined in the Dependency table.
4. **Externalise secrets** using Ansible Vault and update playbooks accordingly.
5. **Update Vagrantfile** to use the `ansible` provisioner and point to the new playbook.
6. **Run iterative tests** – provision a fresh VM, apply the roles, and verify functional parity with the original Chef run.
7. **Document** the migration process, known caveats, and hand‑off checklist for the operations team.

---

*Prepared by the Migration Planning Agent*