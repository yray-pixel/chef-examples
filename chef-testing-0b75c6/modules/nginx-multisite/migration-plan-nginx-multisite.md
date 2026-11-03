---
source-path: cookbooks/nginx-multisite
---

# Migration Plan: nginx‑multisite

**TLDR**: This cookbook provisions a hardened Nginx web‑server that hosts three HTTPS sites (`test.cluster.local`, `ci.cluster.local`, `status.cluster.local`). It installs security utilities (fail2ban, ufw, SSH hardening), creates self‑signed certificates for each site, deploys site document roots with a default `index.html`, and configures Nginx virtual‑host files. All steps are deterministic – the three sites are explicitly enumerated in the Chef code.

## Service Type and Instances

**Service Type**: Web Server (Nginx)

**Configured Instances**:
- **test.cluster.local**: HTTPS site  
  - Location/Path: Document root `/opt/server/test`  
  - Port/Socket: **80** (HTTP) and **443** (HTTPS)  
  - Key Config: SSL enabled – certificate `/etc/ssl/certs/test.cluster.local.crt`, key `/etc/ssl/private/test.cluster.local.key`
- **ci.cluster.local**: HTTPS site  
  - Location/Path: Document root `/opt/server/ci`  
  - Port/Socket: **80** (HTTP) and **443** (HTTPS)  
  - Key Config: SSL enabled – certificate `/etc/ssl/certs/ci.cluster.local.crt`, key `/etc/ssl/private/ci.cluster.local.key`
- **status.cluster.local**: HTTPS site  
  - Location/Path: Document root `/opt/server/status`  
  - Port/Socket: **80** (HTTP) and **443** (HTTPS)  
  - Key Config: SSL enabled – certificate `/etc/ssl/certs/status.cluster.local.crt`, key `/etc/ssl/private/status.cluster.local.key`

## File Structure

**MANDATORY: Preserve this section from the original plan.**
```
### Recipes
recipes/default.rb
recipes/security.rb
recipes/nginx.rb
recipes/ssl.rb
recipes/sites.rb

### Templates
templates/default/fail2ban.jail.local.erb
templates/default/nginx.conf.erb
templates/default/security.conf.erb
templates/default/site.conf.erb
templates/default/sysctl-security.conf.erb

### Attributes
attributes/default.rb

### Files (static content)
# The cookbook copies an index.html for each site.
# The source file lives under the cookbook’s files/ directory (not listed in the tree),
# e.g.: files/default/<site_folder>/index.html

### Providers
*No custom resources are used in the execution flow.*
```

## Module Explanation

The cookbook performs operations in this order (all paths are full paths from the cookbook root `cookbooks/nginx-multisite`).

1. **cookbooks/nginx-multisite/recipes/default.rb**  
   - Includes the following recipes in exact order:  
     1. `nginx-multisite::security` → `cookbooks/nginx-multisite/recipes/security.rb`  
     2. `nginx-multisite::nginx` → `cookbooks/nginx-multisite/recipes/nginx.rb`  
     3. `nginx-multisite::ssl` → `cookbooks/nginx-multisite/recipes/ssl.rb`  
     4. `nginx-multisite::sites` → `cookbooks/nginx-multisite/recipes/sites.rb`

2. **cookbooks/nginx-multisite/recipes/security.rb**  
   - `package[fail2ban ufw]` → `:install`  
   - `service[fail2ban]` → `:enable, :start`  
   - `template[/etc/fail2ban/jail.local]` → source `fail2ban.jail.local.erb`  
   - `execute[ufw_default_deny]` → `ufw --force default deny`  
   - `execute[ufw_allow_ssh]` → `ufw allow ssh`  
   - `execute[ufw_allow_http]` → `ufw allow http`  
   - `execute[ufw_allow_https]` → `ufw allow https`  
   - `execute[ufw_enable]` → `ufw --force enable`  
   - `template[/etc/sysctl.d/99-security.conf]` → source `sysctl-security.conf.erb`  
   - `execute[reload_sysctl]` → `sysctl -p /etc/sysctl.d/99-security.conf` (action `:nothing`, notified by the template)  
   - Conditional `execute[disable root login]` (runs if `node['security']['ssh']['disable_root']` is true) – disables SSH root login.  
   - Conditional `execute[disable password auth]` (runs if `node['security']['ssh']['password_auth'] == false`) – disables password authentication.  
   - `service[ssh]` → `:nothing` (notification target).

3. **cookbooks/nginx-multisite/recipes/nginx.rb**  
   - `package[nginx]` → `:install`  
   - `template[/etc/nginx/nginx.conf]` → source `nginx.conf.erb`, mode `0644`  
   - `template[/etc/nginx/conf.d/security.conf]` → source `security.conf.erb`, mode `0644`  
   - `service[nginx]` → `:enable, :start`  
   - **Loop over `node['nginx']['sites']` (3 items)**:  
     - **test.cluster.local**  
       - `directory[/opt/server/test]` → mode `0755`, owner `www-data`, action `:create`  
       - `cookbook_file[/opt/server/test/index.html]` → source `test/index.html`, mode `0644`  
     - **ci.cluster.local**  
       - `directory[/opt/server/ci]` → mode `0755`, owner `www-data`, action `:create`  
       - `cookbook_file[/opt/server/ci/index.html]` → source `ci/index.html`, mode `0644`  
     - **status.cluster.local**  
       - `directory[/opt/server/status]` → mode `0755`, owner `www-data`, action `:create`  
       - `cookbook_file[/opt/server/status/index.html]` → source `status/index.html`, mode `0644`

4. **cookbooks/nginx-multisite/recipes/ssl.rb**  
   - `package[%w[openssl ca-certificates]]` → `:install`  
   - `group[ssl-cert]` → `:create`  
   - `directory[/etc/ssl/certs]` → mode `0755`, action `:create`  
   - `directory[/etc/ssl/private]` → mode `0710`, action `:create`  
   - **Loop over `node['nginx']['sites']` (3 items)**:  
     - **test.cluster.local** – `execute[generate-ssl-cert-test.cluster.local]` creates  
       - Certificate: `/etc/ssl/certs/test.cluster.local.crt`  
       - Key: `/etc/ssl/private/test.cluster.local.key` (chmod 640, owned by `root:ssl-cert`)  
     - **ci.cluster.local** – `execute[generate-ssl-cert-ci.cluster.local]` creates  
       - Certificate: `/etc/ssl/certs/ci.cluster.local.crt`  
       - Key: `/etc/ssl/private/ci.cluster.local.key`  
     - **status.cluster.local** – `execute[generate-ssl-cert-status.cluster.local]` creates  
       - Certificate: `/etc/ssl/certs/status.cluster.local.crt`  
       - Key: `/etc/ssl/private/status.cluster.local.key`

5. **cookbooks/nginx-multisite/recipes/sites.rb**  
   - **Loop over `node['nginx']['sites']` (3 items)**:  
     - **test.cluster.local**  
       - `template[/etc/nginx/sites-available/test.cluster.local]` → source `site.conf.erb`, variables `server_name`, `document_root`, `ssl_enabled`, `cert_file`, `key_file`; mode `0644`  
       - `link[/etc/nginx/sites-enabled/test.cluster.local]` → creates symlink to the file in `sites-available`  
     - **ci.cluster.local**  
       - `template[/etc/nginx/sites-available/ci.cluster.local]` (same variables)  
       - `link[/etc/nginx/sites-enabled/ci.cluster.local]`  
     - **status.cluster.local**  
       - `template[/etc/nginx/sites-available/status.cluster.local]` (same variables)  
       - `link[/etc/nginx/sites-enabled/status.cluster.local]`  
   - `file[/etc/nginx/sites-enabled/default]` → `:delete` (removes the default site).

## Dependencies

- **External cookbook dependencies**: None declared in metadata.  
- **System package dependencies**: `nginx`, `fail2ban`, `ufw`, `openssl`, `ca-certificates`.  
- **Service dependencies**: `nginx` (systemd), `fail2ban` (systemd), `ssh` (systemd – notified by security recipe).

## Credentials

**Detection Summary**: No credentials or secrets were detected in this cookbook.  
All configuration values are non‑sensitive. SSL certificates are self‑signed and generated on‑the‑fly; no private keys are sourced from external stores.

## Checks for the Migration

**Files to verify**
- `/etc/nginx/nginx.conf`
- `/etc/nginx/conf.d/security.conf`
- `/etc/nginx/sites-available/test.cluster.local`
- `/etc/nginx/sites-available/ci.cluster.local`
- `/etc/nginx/sites-available/status.cluster.local`
- `/etc/nginx/sites-enabled/test.cluster.local` (symlink)
- `/etc/nginx/sites-enabled/ci.cluster.local` (symlink)
- `/etc/nginx/sites-enabled/status.cluster.local` (symlink)
- `/etc/fail2ban/jail.local`
- `/etc/sysctl.d/99-security.conf`
- `/etc/ssl/certs/test.cluster.local.crt`
- `/etc/ssl/certs/ci.cluster.local.crt`
- `/etc/ssl/certs/status.cluster.local.crt`
- `/etc/ssl/private/test.cluster.local.key`
- `/etc/ssl/private/ci.cluster.local.key`
- `/etc/ssl/private/status.cluster.local.key`
- Document roots: `/opt/server/test/index.html`, `/opt/server/ci/index.html`, `/opt/server/status/index.html`

**Service endpoints to check**
| Service | Port(s) | Expected listener |
|---------|---------|-------------------|
| Nginx (HTTP) | 80 | `0.0.0.0:80` (all three sites) |
| Nginx (HTTPS) | 443 | `0.0.0.0:443` (all three sites) |
| Fail2ban | – | `systemctl status fail2ban` (active) |
| UFW | – | `ufw status` (ssh, http, https allowed; default deny) |
| SSH | 22 | `sshd` running, root login disabled, password auth disabled |

**Templates rendered**
| Template | Rendered for | Destination(s) |
|----------|--------------|----------------|
| `fail2ban.jail.local.erb` | once | `/etc/fail2ban/jail.local` |
| `nginx.conf.erb` | once | `/etc/nginx/nginx.conf` |
| `security.conf.erb` | once | `/etc/nginx/conf.d/security.conf` |
| `site.conf.erb` | three times | `/etc/nginx/sites-available/<site>.cluster.local` (one per site) |
| `sysctl-security.conf.erb` | once | `/etc/sysctl.d/99-security.conf` |

## Pre‑flight Checks
```bash
# -------------------------------------------------
# 1. Verify package installation
# -------------------------------------------------
dpkg -l | grep -E 'nginx|fail2ban|ufw|openssl|ca-certificates'

# -------------------------------------------------
# 2. Systemd service status
# -------------------------------------------------
systemctl status nginx
systemctl status fail2ban
systemctl status ssh

# -------------------------------------------------
# 3. Firewall (UFW) configuration
# -------------------------------------------------
ufw status verbose
# Expected: Default deny, allow ssh, http, https

# -------------------------------------------------
# 4. SSH hardening
# -------------------------------------------------
grep -E 'PermitRootLogin|PasswordAuthentication' /etc/ssh/sshd_config
# Expected: PermitRootLogin no, PasswordAuthentication no

# -------------------------------------------------
# 5. Fail2ban configuration
# -------------------------------------------------
cat /etc/fail2ban/jail.local | grep -i enabled
# Expected: enabled = true for relevant jails

# -------------------------------------------------
# 6. Sysctl security settings
# -------------------------------------------------
sysctl -a | grep -E 'ipv4|tcp_syncookies|net.ipv4.conf.all.rp_filter'
# Values should match the template

# -------------------------------------------------
# 7. Verify SSL certificates for each site
# -------------------------------------------------
for site in test ci status; do
  openssl x509 -in /etc/ssl/certs/${site}.cluster.local.crt -noout -text | grep "Subject:"
  ls -l /etc/ssl/private/${site}.cluster.local.key
done

# -------------------------------------------------
# 8. Verify Nginx virtual‑host files and symlinks
# -------------------------------------------------
for site in test ci status; do
  nginx -t -c /etc/nginx/nginx.conf && echo "NGINX config syntax OK"
  test -L /etc/nginx/sites-enabled/${site}.cluster.local && echo "${site} symlink OK"
done

# -------------------------------------------------
# 9. Check document roots and default pages
# -------------------------------------------------
for site in test ci status; do
  curl -I http://${site}.cluster.local/        # HTTP (port 80)
  curl -k -I https://${site}.cluster.local/    # HTTPS (port 443, -k to accept self‑signed)
done

# -------------------------------------------------
# 10. Verify that the default site is removed
# -------------------------------------------------
test ! -e /etc/nginx/sites-enabled/default && echo "Default site removed"
```