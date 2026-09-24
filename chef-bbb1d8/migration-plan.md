# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx setup with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting 3 Chef cookbooks with their recipes, templates, and attributes to equivalent Ansible roles and playbooks.

**Estimated Timeline**: 2-3 weeks for a complete migration, with an additional 1 week for testing and validation.

**Complexity**: Medium - The repository has a moderate number of cookbooks with straightforward configurations, but includes security hardening and SSL certificate management that will require careful migration.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled subdomains, security hardening (fail2ban, ufw), and site-specific configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw), custom Nginx configuration

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Dependency management for Chef cookbooks - lists both local and external dependencies with version constraints
- `solo.json`: Chef run list and node attributes configuration - defines the sites to be configured and security settings
- `solo.rb`: Chef Solo configuration file
- `Vagrantfile`: Defines the development environment using Vagrant with Fedora 42
- `vagrant-provision.sh`: Shell script to provision the Vagrant VM with Chef

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0) based on cookbook metadata, but the Vagrantfile uses Fedora 42
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or community.general.nginx_* modules
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or package installation tasks with custom configuration

### Security Considerations

- **Firewall Configuration**: The current setup uses UFW for firewall management - migrate to appropriate Ansible firewall module (ufw_rule or firewalld depending on target OS)
- **Fail2ban Configuration**: Migrate fail2ban configuration templates to Ansible templates
- **SSH Hardening**: Current configuration disables root login and password authentication - implement using Ansible's openssh_* modules
- **SSL Certificate Management**: Self-signed certificates are generated for each site - implement using Ansible's openssl_* modules
- **Vault/secrets management**:
  - Redis password is hardcoded in the cache cookbook's default recipe
  - PostgreSQL credentials are hardcoded in the fastapi-tutorial cookbook's default recipe
  - No external secret management system is used

### Technical Challenges

- **Multi-site Nginx Configuration**: The current setup dynamically creates site configurations based on node attributes - implement using Ansible loops and templates
- **SSL Certificate Generation**: Self-signed certificates are generated for each site - implement using Ansible's openssl_certificate module
- **Service Dependencies**: The FastAPI application depends on PostgreSQL - ensure proper ordering in Ansible playbooks
- **Idempotency**: Ensure all custom shell commands are replaced with idempotent Ansible modules

### Migration Order

1. **nginx-multisite cookbook** (moderate complexity, foundation for other services)
   - Start with basic Nginx installation and configuration
   - Add security hardening (fail2ban, ufw, sysctl)
   - Implement SSL certificate generation
   - Configure multi-site setup

2. **cache cookbook** (low complexity)
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial cookbook** (moderate complexity)
   - Implement PostgreSQL installation and configuration
   - Set up Python environment and application deployment
   - Configure systemd service

### Assumptions

1. The target environment will continue to be either Ubuntu (>= 18.04) or CentOS (>= 7.0), with Fedora 42 for development
2. The same network configuration and port mappings will be maintained
3. Self-signed certificates are acceptable for development (no Let's Encrypt or other CA integration required)
4. No changes to the application code or database schema are needed
5. The current hardcoded secrets will be migrated to Ansible Vault for improved security
6. The same VM resources (2GB RAM, 2 CPUs) will be sufficient for the Ansible-managed environment