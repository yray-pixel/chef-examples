# MIGRATION FROM CHEF TO ANSIBLE

**Executive Summary**
The repository is a Chef-based infrastructure codebase consisting of three primary cookbooks (`cache`, `fastapi-tutorial`, and `nginx-multisite`) plus a handful of supporting provisioning scripts and a Berksfile that pins external community cookbooks.  The scope of the migration is modest – roughly 1 000 lines of Chef DSL across three logical services – but it touches a variety of technologies (memcached, Redis, PostgreSQL, Python virtual‑environment, systemd, Nginx with multi‑site SSL).  A phased, low‑risk migration can be completed in **3 – 4 weeks** by a small team (1‑2 engineers) with expertise in both Chef and Ansible.

---

## Module Migration Plan

This repository contains **Chef cookbooks** that need individual migration planning:

### MODULE INVENTORY

- **cache**
  - **Description**: Sets up caching layer – installs `memcached`, configures a secured Redis instance (password, log directory, custom config tweaks), and ensures services are enabled.
  - **Path**: `cookbooks/cache`
  - **Technology**: Chef
  - **Key Features**:
    - Includes external `memcached` and `redisio` cookbooks.
    - Sets Redis password (`redis_secure_password_123`).
    - Performs a ruby‑block hack to clean up default Redis config entries.
    - Creates `/var/log/redis` directory with proper ownership.

- **fastapi-tutorial**
  - **Description**: Deploys a demo FastAPI application. Installs Python 3, system packages, clones a Git repo, builds a virtualenv, installs Python deps, creates a PostgreSQL database/user, writes a `.env` file with plaintext credentials, installs a systemd service for the app.
  - **Path**: `cookbooks/fastapi-tutorial`
  - **Technology**: Chef
  - **Key Features**:
    - Packages: `python3`, `python3-pip`, `python3-venv`, `git`, `postgresql` etc.
    - PostgreSQL user/password (`fastapi_password`).
    - Systemd unit file `fastapi-tutorial.service`.
    - Environment file containing DB connection string.

- **nginx-multisite**
  - **Description**: Configures Nginx as a multi‑site reverse‑proxy with SSL termination. Provides separate recipes for security hardening, base Nginx install, SSL cert handling, and per‑site document roots.
  - **Path**: `cookbooks/nginx-multisite`
  - **Technology**: Chef
  - **Key Features**:
    - Installs `nginx` package.
    - Templates: `nginx.conf.erb`, `security.conf.erb`, `site.conf.erb` (per‑site config).
    - Manages site files under `files/default/*` (static `index.html` examples).
    - Enables and reloads Nginx service.
    - Relies on attribute `node['nginx']['sites']` for dynamic site creation.

**CRITICAL PATH VERIFICATION**
All modules above were discovered in the repository tree and verified by reading their `metadata.rb` and primary recipe files.

---

### Infrastructure Files

- `Berksfile` – pins external community cookbooks (`nginx`, `memcached`, `redisio`).  These will need Ansible Galaxy equivalents or custom role implementation.
- `Vagrantfile` – defines a Vagrant VM used for local testing; may translate to an Ansible‑compatible Vagrant provisioner or be replaced by a CI/CD pipeline.
- `solo.json` / `solo.rb` – Chef Solo configuration; will be discarded once Ansible playbooks replace the Chef run.
- `vagrant-provision.sh` – Bash provisioning script executed by Vagrant; may contain useful inline commands to be ported into Ansible tasks.

---

## Target Details

- **Operating System**: The cookbooks declare support for Ubuntu ≥ 18.04 and CentOS ≥ 7.  The Vagrant environment (looked at `Vagrantfile`) defaults to Ubuntu 20.04, so the primary target will be **Ubuntu 22.04 LTS** (most recent LTS with long‑term support).  All package names and systemd interactions are compatible with this OS.
- **Virtual Machine Technology**: Vagrant (VirtualBox provider).  No cloud‑specific resources are present.
- **Cloud Platform**: Not specified – the code is cloud‑agnostic.

---

## Migration Approach

### Key Dependencies to Address

| Dependency | Current Source | Ansible Equivalent / Migration Strategy |
|------------|----------------|-------------------------------------------|
| `nginx` (Chef cookbook) | Berksfile (`cookbook 'nginx', '~> 12.0'`) | Use the **`geerlingguy.nginx`** role from Ansible Galaxy or create a custom role that mirrors the existing templates. |
| `memcached` | Berksfile (`cookbook 'memcached'`) | Use **`geerlingguy.memcached`** role or a custom role; ensure identical configuration flags. |
| `redisio` | Berksfile (`cookbook 'redisio', '~> 7.2.4'`) | Use **`geerlingguy.redis`** role; map the password attribute to `redis_password`. |
| `postgresql` (system packages) | Installed via Chef `package` resource | Use Ansible `apt` module; systemd service handling via `systemd` module. |
| Python virtual‑env & pip | Chef `execute` resources | Use Ansible `pip` and `python` modules; `ansible.builtin.command` for `python3 -m venv` if needed. |
| Systemd services | Chef `service` and `file` resources | Use Ansible `systemd` module with `daemon_reload` handler. |
| Git checkout | Chef `git` resource | Use Ansible `git` module. |
| Templates (`*.erb`) | Chef `template` resource | Convert to Jinja2 templates (`*.j2`). |

### Security Considerations

- **Plaintext passwords** appear in:
  - `cookbooks/cache/recipes/default.rb` – Redis password `redis_secure_password_123`.
  - `cookbooks/fastapi-tutorial/recipes/default.rb` – PostgreSQL user `fastapi` password `fastapi_password` stored in the generated `.env` file.
  - Both will need to be externalized to **Ansible Vault** or a centralized secrets manager (HashiCorp Vault, AWS Secrets Manager, etc.).
- **SSL/TLS**: The `nginx-multisite` cookbook expects SSL certificates (likely placed via the `ssl.rb` recipe). Verify where cert files reside; migrate to Ansible `copy`/`file` tasks with proper permissions, or consider using `acme_certificate` module for automated Let's Encrypt handling.
- **File permissions**: Ensure all files created by Ansible mirror the original modes (`0644`, `0755`, etc.) and ownership (`root`, `redis`, `www-data`).
- **Ruby‑block hack**: Direct file manipulation of Redis config should be expressed as an Ansible `lineinfile` or `replace` task to maintain idempotency.

### Technical Challenges

1. **Chef DSL vs Ansible YAML** – Translating `node.default` attribute assignments to Ansible `vars` and `defaults/main.yml`.  Strategy: create role defaults that reflect the original attribute structure.
2. **Dynamic site generation** – The Nginx cookbook iterates over `node['nginx']['sites']`.  In Ansible this will be a `with_dict` loop over a variable defined in group/host vars or inventory.
3. **Ruby‑block configuration edits** – Need to ensure Ansible idempotent replacements; test on a clean VM to verify resulting `redis.conf` matches expectations.
4. **Systemd service templates** – The multi‑line heredoc used in Chef `file` resource will be converted to a Jinja2 template (`fastapi-tutorial.service.j2`).
5. **Testing** – Existing Vagrant/Chef run provides a test harness.  We'll retain it for regression testing while developing Ansible playbooks.

### Migration Order

1. **Infrastructure Bootstrap** – Convert `Vagrantfile`/`vagrant-provision.sh` to an Ansible playbook that installs required base packages (`git`, `python3`, `apt-transport-https`).  This establishes a clean execution environment.
2. **Cache Layer (`cache` cookbook)** – Small, self‑contained; migrate Redis and Memcached first because other services (FastAPI) depend on Redis being available for caching.
3. **Nginx Multi‑Site (`nginx-multisite` cookbook)** – Medium complexity; implement using Ansible role with templates, then validate site provisioning.
4. **FastAPI Application (`fastapi-tutorial` cookbook)** – Highest complexity due to multiple moving parts (Python venv, DB, systemd, secrets).  Perform after underlying services are verified.

### Assumptions

- The repository does **not** include any Puppet, Salt, PowerShell, or DSC artifacts; all IaC is Chef‑based.
- No external secret management system is currently used; all credentials are hard‑coded in the recipes.
- The target environment will remain on a Linux OS (Ubuntu 22.04).  If CentOS support is needed, package names will be adjusted accordingly.
- The external community cookbooks (`nginx`, `memcached`, `redisio`) have feature‑parity with the Ansible Galaxy roles we plan to use.
- The Vagrant box used for testing mirrors the production OS version closely enough that Ansible playbooks validated locally can be promoted without major OS‑specific changes.

---

## Next Steps
1. **Create Ansible role skeletons** (`cache`, `nginx_multisite`, `fastapi_tutorial`).
2. **Externalize secrets** to Ansible Vault and update the repository with encrypted files.
3. **Port templates** from ERB to Jinja2.
4. **Write playbooks** that orchestrate role execution in the order defined above.
5. **Perform regression testing** using the existing Vagrant environment.
6. **Document** the migration process and hand‑off to the operations team.

---

*Prepared by the Migration Planning Agent on 2026‑11‑10.*