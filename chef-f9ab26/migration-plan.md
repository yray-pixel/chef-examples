# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server setup with FastAPI application backend and caching services. The migration to Ansible will involve converting 3 Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The repository has a clear structure with well-defined cookbooks
- External dependencies on community cookbooks need to be replaced with Ansible Galaxy roles
- Security configurations need careful migration
- SSL certificate handling requires special attention

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and site configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw)
    - Entry Point: cookbooks/nginx-multisite/recipes/default.rb (verified)

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management
    - Entry Point: cookbooks/fastapi-tutorial/recipes/default.rb (verified)

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration
    - Entry Point: cookbooks/cache/recipes/default.rb (verified)

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks. Lists both local and external cookbook dependencies.
- `solo.json`: Chef configuration data with run list and node attributes for Nginx sites, SSL, and security settings.
- `solo.rb`: Chef Solo configuration file specifying cookbook paths and log settings.
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef.
- `Vagrantfile`: Vagrant configuration for development environment using Fedora 42.

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>=18.04) and CentOS (>=7.0) based on cookbook metadata, but the Vagrantfile uses Fedora 42.
- **Virtual Machine Technology**: Vagrant with libvirt provider.
- **Cloud Platform**: Not specified, appears to be designed for on-premises or local development.

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role from Galaxy or custom role
- **memcached (~> 6.0)**: Replace with Ansible memcached role from Galaxy
- **redisio (~> 7.2.4)**: Replace with Ansible redis role from Galaxy

### Security Considerations

- **SSL Certificate Management**: 
  - Self-signed certificates are generated for each site
  - Migration should maintain the same level of security with proper file permissions
  - Consider integrating with Ansible's crypto modules for certificate generation

- **Firewall Configuration**: 
  - UFW firewall rules need to be migrated to equivalent Ansible ufw module tasks
  - Default deny policy with specific allows for SSH, HTTP, and HTTPS

- **Fail2ban Integration**:
  - Fail2ban configuration needs to be migrated to Ansible
  - Custom jail configuration template needs to be preserved

- **SSH Hardening**:
  - Root login disabled
  - Password authentication disabled
  - These settings need to be maintained in the Ansible playbooks

- **Vault/secrets management**:
  - Redis password is hardcoded in the recipe (`redis_secure_password_123`)
  - PostgreSQL credentials are hardcoded in the FastAPI recipe (`fastapi:fastapi_password`)
  - These should be moved to Ansible Vault or other secure storage

### Technical Challenges

- **Multi-site Configuration**: The dynamic generation of Nginx site configurations based on node attributes needs to be replicated in Ansible using templates and variables.

- **Service Dependencies**: The FastAPI application depends on PostgreSQL, and the Nginx sites depend on the SSL certificates. These dependencies need to be maintained in the Ansible playbook ordering.

- **Custom Configuration Hacks**: The Redis configuration has a custom Ruby block to modify configuration files after installation. This needs a clean implementation in Ansible.

- **Python Environment Management**: The FastAPI application uses a Python virtual environment that needs to be properly managed in Ansible.

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Split into separate roles for nginx, ssl, and security

2. **cache** (Priority 2)
   - Independent service but required by applications
   - Separate into memcached and redis roles

3. **fastapi-tutorial** (Priority 3)
   - Application deployment that depends on database and potentially cache services
   - Create roles for Python application deployment and PostgreSQL

### Assumptions

1. The target environment will continue to use the same operating systems (Ubuntu/CentOS/Fedora).
2. Self-signed certificates are acceptable for the migrated solution (not using Let's Encrypt or other CA).
3. The same security policies should be applied in the Ansible version.
4. The Vagrant development environment should be preserved with equivalent functionality.
5. No changes to the application code or database schema are required.
6. The current hardcoded credentials will be replaced with Ansible Vault secured variables.
7. The migration will maintain the same directory structure for deployed applications and configuration files.