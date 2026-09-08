# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting 3 Chef cookbooks with their recipes, templates, and attributes to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The repository has a moderate number of cookbooks with clear responsibilities
- Security configurations need careful attention during migration
- Self-signed SSL certificates and multi-site configuration require proper handling

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and firewall configuration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, fail2ban integration, UFW firewall setup, security hardening

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, PostgreSQL database creation, systemd service configuration, Git repository deployment

### Infrastructure Files

- `Berksfile`: Dependency management file listing cookbook dependencies and their sources
- `solo.json`: Chef configuration file with run list and node attributes
- `solo.rb`: Chef configuration file for Chef Solo
- `Vagrantfile`: Defines the development VM environment using Vagrant
- `vagrant-provision.sh`: Shell script to provision the Vagrant VM with Chef

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (primary) with support for Ubuntu 18.04+ and CentOS 7+ mentioned in cookbook metadata
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or nginx_core module
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package module
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or package module with custom configuration

### Security Considerations

- **SSL Certificate Management**: Self-signed certificates are generated for each site - migrate to Ansible crypto modules
- **Firewall Configuration**: UFW rules need to be migrated to Ansible ufw module or firewalld for Fedora
- **fail2ban Integration**: Configuration needs to be migrated to Ansible templates
- **SSH Hardening**: SSH configuration (disabling root login, password authentication) needs careful migration
- **Vault/secrets management**:
  - Redis password is hardcoded in the cache cookbook
  - PostgreSQL credentials are hardcoded in the FastAPI cookbook
  - Consider migrating to Ansible Vault for secure credential storage

### Technical Challenges

- **Multi-site Nginx Configuration**: The dynamic generation of multiple virtual hosts with SSL needs careful implementation in Ansible
- **Self-signed Certificate Generation**: The SSL certificate generation logic needs to be replicated using Ansible crypto modules
- **Service Dependencies**: Ensuring proper ordering of service deployments (database before application, etc.)
- **Idempotent Security Configurations**: Ensuring security configurations are applied idempotently

### Migration Order

1. **cache cookbook**: Start with the simplest cookbook that installs and configures Memcached and Redis
2. **nginx-multisite cookbook**: Next, migrate the Nginx configuration with its security features
3. **fastapi-tutorial cookbook**: Finally, migrate the application deployment that depends on the other services

### Assumptions

1. The target environment will continue to be Fedora-based, though the cookbooks support Ubuntu and CentOS as well
2. The self-signed certificates approach is acceptable for the migrated solution (vs. using Let's Encrypt or other CA)
3. The current security configurations are appropriate and should be maintained in the Ansible implementation
4. The current directory structure for web content will be maintained
5. The Vagrant development environment will be maintained but updated to use Ansible provisioning
6. No CI/CD pipeline integration is required as part of the migration
7. The FastAPI application source will continue to be pulled from the same Git repository