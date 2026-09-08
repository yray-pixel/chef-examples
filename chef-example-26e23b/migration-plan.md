# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting 3 Chef cookbooks with their recipes, templates, and attributes to equivalent Ansible roles and playbooks.

**Estimated Timeline:** 2-3 weeks for a complete migration, with an additional 1 week for testing and validation.

**Complexity:** Medium - The repository has a moderate number of cookbooks with straightforward configurations, but includes security hardening and SSL certificate management that will require careful migration.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled subdomains, security hardening, and site configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, fail2ban integration, UFW firewall configuration, security hardening

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

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies
- `solo.json`: Chef configuration file containing the run list and node attributes
- `solo.rb`: Chef configuration file for Chef Solo
- `Vagrantfile`: Defines the development environment using Vagrant with Fedora 42
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant VM with Chef

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0), with Fedora 42 used in the Vagrant development environment
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or community.general.nginx_* modules
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or package installation tasks

### Security Considerations

- **Fail2ban configuration**: Migrate fail2ban jail configuration template to Ansible template
- **UFW firewall rules**: Use Ansible's ufw module to configure firewall rules
- **SSH hardening**: Migrate SSH configuration (disable root login, password authentication) using Ansible's lineinfile or template module
- **Sysctl security settings**: Migrate sysctl security configuration using Ansible's sysctl module
- **Vault/secrets management**:
  - Redis authentication password hardcoded in the cache cookbook
  - PostgreSQL database credentials hardcoded in the FastAPI cookbook
  - SSL certificates and private keys generated and managed by the nginx-multisite cookbook
  - Consider migrating to Ansible Vault for secure credential storage

### Technical Challenges

- **SSL Certificate Management**: The current implementation generates self-signed certificates. Consider using Ansible's openssl_* modules or integrating with Let's Encrypt via the community.crypto collection
- **Multi-site Configuration**: The dynamic generation of Nginx site configurations based on node attributes will need to be carefully migrated to Ansible's template system
- **Service Dependencies**: Ensuring proper ordering of service installations and configurations, particularly for the FastAPI application which depends on PostgreSQL

### Migration Order

1. **cache cookbook** (low complexity): Simple installation and configuration of Memcached and Redis
2. **nginx-multisite cookbook** (medium complexity): Nginx installation, configuration, and security hardening
3. **fastapi-tutorial cookbook** (high complexity): Application deployment with database dependencies

### Assumptions

1. The target environment will continue to be either Ubuntu (>= 18.04) or CentOS (>= 7.0)
2. Self-signed certificates are acceptable for development, but production may require integration with a certificate authority
3. The current security configurations are appropriate and should be maintained in the Ansible implementation
4. The FastAPI application source will continue to be pulled from the same Git repository
5. The current Redis and PostgreSQL passwords are development credentials and will be replaced with secure passwords in production
6. The Vagrant development environment will be maintained for testing the Ansible playbooks