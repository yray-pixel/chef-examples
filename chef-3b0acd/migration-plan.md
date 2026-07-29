# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site web server with caching services and a FastAPI application. The migration to Ansible will involve converting three Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks. The estimated timeline for this migration is 3-4 weeks, with moderate complexity due to the interdependencies between services and security configurations.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and fail2ban integration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security headers, firewall configuration

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Git repository deployment, Python virtual environment, PostgreSQL database setup, systemd service configuration

### Infrastructure Files

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies
- `solo.json`: Chef Solo configuration file with run list and node attributes
- `solo.rb`: Chef Solo configuration file with file paths and log settings
- `Vagrantfile`: Vagrant configuration for local development using Fedora 42
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be designed for on-premises deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible Galaxy role `geerlingguy.nginx` or create custom Ansible role
- **memcached (~> 6.0)**: Replace with Ansible Galaxy role `geerlingguy.memcached`
- **redisio (~> 7.2.4)**: Replace with Ansible Galaxy role `geerlingguy.redis` or DavidWittman.redis

### Security Considerations

- **SSL Certificate Management**: 
  - Migration approach: Use Ansible `openssl_*` modules for certificate generation
  - Consider integration with Let's Encrypt using `geerlingguy.certbot` role

- **Firewall Configuration (UFW)**:
  - Migration approach: Use Ansible `ufw` module to configure firewall rules

- **Fail2ban Configuration**:
  - Migration approach: Use Ansible to template fail2ban configuration files

- **SSH Hardening**:
  - Migration approach: Use Ansible to configure SSH security settings or consider `dev-sec.ssh-hardening` role

- **Vault/secrets management**:
  - Redis password is hardcoded in the cache cookbook
  - PostgreSQL credentials are hardcoded in the fastapi-tutorial cookbook
  - Migration approach: Use Ansible Vault to encrypt sensitive values

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - Description: The current implementation uses Chef templates to generate multiple virtual host configurations
  - Mitigation strategy: Create Ansible templates with similar logic, using Ansible loops to iterate through site configurations

- **Service Interdependencies**:
  - Description: The FastAPI application depends on PostgreSQL, and the web server depends on the application being available
  - Mitigation strategy: Use Ansible handlers and the `notify` mechanism to ensure proper service restart order

- **SSL Certificate Generation**:
  - Description: Self-signed certificates are generated for development
  - Mitigation strategy: Use Ansible's `openssl_certificate` module with similar parameters

### Migration Order

1. **cache** (low risk, moderate value)
   - Simple configuration of standard services
   - Few custom configurations

2. **nginx-multisite** (moderate complexity)
   - Core infrastructure component
   - Contains security configurations
   - Multiple templates and configurations

3. **fastapi-tutorial** (high complexity, dependencies)
   - Depends on PostgreSQL
   - Involves application deployment and configuration
   - Requires environment setup

### Assumptions

1. The target environment will continue to be Fedora-based systems
2. Self-signed certificates are acceptable for development environments
3. The same security hardening requirements will apply in the new environment
4. The FastAPI application repository will remain available at the specified URL
5. The current network configuration (ports, IP addresses) will remain the same
6. No changes to the application architecture are planned during migration
7. Redis and Memcached configurations will remain similar
8. PostgreSQL database schema and user permissions will remain unchanged