---
source-path: cookbooks/nginx-multisite
---

# Migration Plan: nginx‑multisite

**TLDR**: This cookbook hardens a Linux host, installs Nginx, Fail2Ban and UFW, creates three HTTPS sites (test.cluster.local, ci.cluster.local, status.cluster.local) with their own document roots and self‑signed certificates, and configures firewall rules and SSH hardening. All resources are deterministic, enabling a straight‑line Ansible migration that mirrors the Chef execution order.

## Service Type and Instances

**Service Type**: Web Server (Nginx)

**Configured Instances**:
- **test.cluster.local**: HTTPS website
  - Location/Path: `/opt/server/test`
  - Port/Socket: `80` (http), `443` (https)
  - Key Config: SSL enabled, certificate `/etc/ssl/certs/test.cluster.local.crt`, key `/etc/ssl/private/test.cluster.local.key`
- **ci.cluster.local**: HTTPS website
  - Location/Path: `/opt/server/ci`
  - Port/Socket: `80` (http), `443` (https)
  - Key Config: SSL enabled, certificate `/etc/ssl/certs/ci.cluster.local.crt`, key `/etc/ssl/private/ci.cluster.local.key`
- **status.cluster.local**: HTTPS website
  - Location/Path: `/opt/server/status`
  - Port/Socket: `80` (http), `443` (https)
  - Key Config: SSL enabled, certificate `/etc/ssl/certs/status.cluster.local.crt`, key `/etc/ssl/private/status.cluster.local.key`

## File Structure

**MANDATORY: Preserve this section from the original plan.**

```
**Recipes**
cookbooks/nginx-multisite/recipes/default.rb
cookbooks/nginx-multisite/recipes/security.rb
cookbooks/nginx-multisite/recipes/nginx.rb
cookbooks/nginx-multisite/recipes/ssl.rb
cookbooks/nginx-multisite/recipes/sites.rb

**Templates**
cookbooks/nginx-multisite/templates/default/fail2ban.jail.local.erb
cookbooks/nginx-multisite/templates/default/nginx.conf.erb
cookbooks/nginx-multisite/templates/default/security.conf.erb
cookbooks/nginx-multisite/templates/default/site.conf.erb
cookbooks/nginx-multisite/templates/default/sysctl-security.conf.erb

**Attributes**
cookbooks/nginx-multisite/attributes/default.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/nginx-multisite/recipes/default.rb`):
   - Includes the four other recipes in sequence: `security.rb`, `nginx.rb`, `ssl.rb`, `sites.rb`.

2. **security** (`cookbooks/nginx-multisite/recipes/security.rb`):
   - Installs packages `fail2ban` and `ufw`.
   - Enables & starts service `fail2ban`.
   - Renders template `fail2ban.jail.local.erb` → `/etc/fail2ban/jail.local`.
   - Executes UFW commands: default deny, allow `ssh`, allow `http`, allow `https`, enable firewall.
   - Renders template `sysctl-security.conf.erb` → `/etc/sysctl.d/99-security.conf` and runs `sysctl -p` when changed.
   - Conditionally disables root SSH login and password authentication based on attributes.
   - Provides a no‑op `service[ssh]` resource for notification targeting.

3. **nginx** (`cookbooks/nginx-multisite/recipes/nginx.rb`):
   - Installs package `nginx`.
   - Renders `nginx.conf.erb` → `/etc/nginx/nginx.conf`.
   - Renders `security.conf.erb` → `/etc/nginx/conf.d/security.conf`.
   - Enables & starts service `nginx`.
   - **Loop over sites (expanded):**
     - `directory[/opt/server/test]` – creates document root for **test.cluster.local**, mode `0755`.
     - `cookbook_file[/opt/server/test/index.html]` – copies static index.html for **test.cluster.local**.
     - `directory[/opt/server/ci]` – creates document root for **ci.cluster.local**, mode `0755`.
     - `cookbook_file[/opt/server/ci/index.html]` – copies static index.html for **ci.cluster.local**.
     - `directory[/opt/server/status]` – creates document root for **status.cluster.local**, mode `0755`.
     - `cookbook_file[/opt/server/status/index.html]` – copies static index.html for **status.cluster.local**.

4. **ssl** (`cookbooks/nginx-multisite/recipes/ssl.rb`):
   - Installs packages `openssl` and `ca-certificates`.
   - Creates group `ssl-cert`.
   - Ensures directories `/etc/ssl/certs` (mode `0755`) and `/etc/ssl/private` (mode `0710`) exist.
   - **Loop over sites (expanded):**
     - `execute[generate-ssl-cert-test.cluster.local]` – creates self‑signed cert `test.cluster.local.crt` and key `test.cluster.local.key`.
     - `execute[generate-ssl-cert-ci.cluster.local]` – creates self‑signed cert `ci.cluster.local.crt` and key `ci.cluster.local.key`.
     - `execute[generate-ssl-cert-status.cluster.local]` – creates self‑signed cert `status.cluster.local.crt` and key `status.cluster.local.key`.

5. **sites** (`cookbooks/nginx-multisite/recipes/sites.rb`):
   - **Loop over sites (expanded):**
     - `template[/etc/nginx/sites-available/test.cluster.local]` – renders `site.conf.erb` for **test.cluster.local** with its variables.
     - `link[/etc/nginx/sites-enabled/test.cluster.local]` – symlinks the above file.
     - `template[/etc/nginx/sites-available/ci.cluster.local]` – renders `site.conf.erb` for **ci.cluster.local**.
     - `link[/etc/nginx/sites-enabled/ci.cluster.local]` – symlinks the above file.
     - `template[/etc/nginx/sites-available/status.cluster.local]` – renders `site.conf.erb` for **status.cluster.local**.
     - `link[/etc/nginx/sites-enabled/status.cluster.local]` – symlinks the above file.
   - Deletes the default site symlink: `file[/etc/nginx/sites-enabled/default]` (action `:delete`).

## Dependencies

**External cookbook dependencies**: None (metadata only lists name/version).  
**System package dependencies**: `nginx`, `fail2ban`, `ufw`, `openssl`, `ca-certificates`.  
**Service dependencies**: `systemd` (or sysvinit) management for `nginx`, `fail2ban`, `ssh`.

## Credentials

**Detection Summary**: 0 credentials detected across 0 files.

**Source**:
  - **Provider**: None detected
  - **URL**: N/A
  - **Path**: N/A

No credentials or secrets were detected in this cookbook. All configuration values appear to be non‑sensitive.

## Checks for the Migration

**Files to verify**:
- `/etc/nginx/nginx.conf`
- `/etc/nginx/conf.d/security.conf`
- `/etc/nginx/sites-available/test.cluster.local`
- `/etc/nginx/sites-available/ci.cluster.local`
- `/etc/nginx/sites-available/status.cluster.local`
- `/etc/nginx/sites-enabled/test.cluster.local`
- `/etc/nginx/sites-enabled/ci.cluster.local`
- `/etc/nginx/sites-enabled/status.cluster.local`
- `/opt/server/test/index.html`
- `/opt/server/ci/index.html`
- `/opt/server/status/index.html`
- `/etc/ssl/certs/test.cluster.local.crt`
- `/etc/ssl/certs/ci.cluster.local.crt`
- `/etc/ssl/certs/status.cluster.local.crt`
- `/etc/ssl/private/test.cluster.local.key`
- `/etc/ssl/private/ci.cluster.local.key`
- `/etc/ssl/private/status.cluster.local.key`
- `/etc/fail2ban/jail.local`
- `/etc/sysctl.d/99-security.conf`
- UFW rules (`ufw status numbered`)

**Service endpoints to check**:
- Nginx: ports **80** (http) and **443** (https) listening.
- Fail2Ban: service `fail2ban` active.
- UFW: default deny, rules for `ssh`, `http`, `https` present.
- SSH: root login disabled, password authentication disabled.

**Templates rendered**:
- `fail2ban.jail.local.erb` → `/etc/fail2ban/jail.local` (1 render)
- `nginx.conf.erb` → `/etc/nginx/nginx.conf` (1 render)
- `security.conf.erb` → `/etc/nginx/conf.d/security.conf` (1 render)
- `site.conf.erb` → `/etc/nginx/sites-available/<site>` (3 renders, one per site)
- `sysctl-security.conf.erb` → `/etc/sysctl.d/99-security.conf` (1 render)

## Pre‑flight Checks:
```bash
# 1. Service status
systemctl status nginx
systemctl status fail2ban
systemctl status ssh

# 2. Firewall (UFW) status
ufw status verbose
# Expect default deny, allow ssh, http, https

# 3. Fail2Ban configuration sanity
grep -i 'enabled' /etc/fail2ban/jail.local | grep true

# 4. Sysctl security settings
sysctl -a | grep -E 'fs.suid_dumpable|kernel.randomize_va_space|net.ipv4.conf.all.rp_filter'

# 5. SSH hardening checks
grep -E '^PermitRootLogin|^PasswordAuthentication' /etc/ssh/sshd_config
# Should report:
# PermitRootLogin no
# PasswordAuthentication no

# 6. Nginx syntax & reload
nginx -t
systemctl reload nginx

# 7. Verify each site reachable over HTTPS (allow self‑signed)
for site in test.cluster.local ci.cluster.local status.cluster.local; do
  curl -k -I https://$site/ | head -n 1
done
# Expect HTTP/1.1 200 OK for each

# 8. Verify document roots contain index.html
for d in /opt/server/test /opt/server/ci /opt/server/status; do
  ls -l $d/index.html
done

# 9. Verify SSL certificate files
for site in test.cluster.local ci.cluster.local status.cluster.local; do
  openssl x509 -noout -subject -in /etc/ssl/certs/${site}.crt
done
```