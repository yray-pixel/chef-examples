# MIGRATION FROM CHEF TO ANSIBLE

This migration plan outlines the process of converting a Chef-based infrastructure to Ansible. The repository contains three Chef cookbooks that manage a multi-site Nginx setup, caching services (Redis and Memcached), and a FastAPI application with PostgreSQL. The estimated timeline for migration is 2-3 weeks, with moderate complexity due to the interdependencies between services.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and site configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw)

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

- `Berksfile`: Manages cookbook dependencies, including external cookbooks from Chef Supermarket (nginx, memcached, redisio)
- `solo.json`: Contains node attributes and run list for Chef Solo, including Nginx site configurations and security settings
- `solo.rb`: Chef Solo configuration file specifying cookbook paths and log settings
- `vagrant-provision.sh`: Bash script for provisioning the Vagrant VM with Chef Solo
- `Vagrantfile`: Defines the Vagrant VM configuration (Fedora 42) with port forwarding and network settings

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0), with Fedora 42 used in Vagrant
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for local development/testing

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or community.general.nginx_* modules
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or package installation tasks

### Security Considerations

- **SSL Certificate Management**: Self-signed certificates are generated for each site; migrate to ansible.builtin.openssl_* modules
- **Firewall Configuration**: UFW rules need to be migrated to ansible.posix.ufw module
- **Fail2ban Configuration**: Custom fail2ban configuration needs to be migrated to Ansible templates
- **SSH Hardening**: SSH configuration (disable root login, password authentication) should be migrated to ansible.posix.ssh_config module
- **Vault/secrets management**:
  - Redis password in cache cookbook: "redis_secure_password_123" (hardcoded)
  - PostgreSQL credentials in fastapi-tutorial cookbook: username "fastapi" with password "fastapi_password" (hardcoded)
  - Total credentials detected: 2 hardcoded passwords

### Technical Challenges

- **Multi-site Nginx Configuration**: The dynamic generation of multiple Nginx sites with SSL will require careful templating in Ansible
- **Service Interdependencies**: The FastAPI application depends on PostgreSQL, and potentially the cache services
- **SSL Certificate Generation**: Self-signed certificate generation logic needs to be replicated in Ansible
- **Security Hardening**: Comprehensive security measures need to be maintained during migration

### Migration Order

1. **cache cookbook** (low complexity, foundational service)
   - Simple package installations and configurations
   - Few dependencies on other services

2. **nginx-multisite cookbook** (moderate complexity)
   - Core web server configuration
   - Multiple site configurations and SSL setup
   - Security hardening components

3. **fastapi-tutorial cookbook** (higher complexity)
   - Application deployment with database
   - Depends on proper web server configuration for access

### Assumptions

1. The target environment will continue to use the same operating systems (Ubuntu/CentOS)
2. Self-signed certificates are acceptable for the migrated environment (production would likely need proper certificates)
3. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git will remain accessible
4. The current security configurations are appropriate for the target environment
5. The Vagrant setup is primarily for development/testing and may not be needed in the final Ansible configuration
6. No custom Chef resources are being used that would require special handling
7. The current hardcoded credentials will be replaced with Ansible Vault variables
8. The directory structure for web content (/var/www/[site]) will remain the same