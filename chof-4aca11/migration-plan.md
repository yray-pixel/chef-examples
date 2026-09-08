# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting 3 Chef cookbooks with their recipes, templates, and attributes to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Small-scale project (3 cookbooks)
- Moderate complexity due to security configurations and SSL setup
- Estimated completion: 2-3 weeks with 1 dedicated engineer

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled subdomains, security hardening, and site configurations
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

- `Berksfile`: Dependency management file listing cookbook dependencies (both local and external from Chef Supermarket)
- `solo.json`: Chef run list and node attributes configuration
- `solo.rb`: Chef Solo configuration file
- `Vagrantfile`: Defines development VM using Fedora 42 with libvirt provider
- `vagrant-provision.sh`: Shell script to provision the Vagrant VM with Chef

### Target Details

Based on the source configuration files:

- **Operating System**: Supports both Ubuntu (>= 18.04) and CentOS (>= 7.0), with Fedora 42 used in Vagrant development environment
- **Virtual Machine Technology**: libvirt (based on Vagrantfile configuration)
- **Cloud Platform**: Not specified, appears to be designed for on-premises or generic cloud deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or nginx_core module
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or package installation tasks

### Security Considerations

- **SSL Certificate Management**: Self-signed certificates are generated for development; migration should maintain this capability while allowing for production certificates
- **Fail2ban Configuration**: Security hardening with fail2ban needs to be migrated to equivalent Ansible tasks
- **UFW Firewall Rules**: Firewall configuration should be migrated to Ansible ufw module or firewalld for RHEL-based systems
- **SSH Hardening**: SSH security configurations (disabling root login, password authentication) should be preserved
- **System Hardening**: sysctl security configurations should be migrated
- **Vault/secrets management**:
  - Redis password is hardcoded in the cache cookbook
  - PostgreSQL credentials are hardcoded in the FastAPI cookbook
  - No external vault integration is present in the current implementation

### Technical Challenges

- **Multi-site Nginx Configuration**: The dynamic generation of multiple site configurations based on node attributes will need careful translation to Ansible variables and templates
- **SSL Certificate Generation**: Self-signed certificate generation logic needs to be preserved while allowing for production certificates
- **Service Dependencies**: Ensuring proper ordering of service installations and configurations (e.g., PostgreSQL before FastAPI application)
- **Platform Compatibility**: Maintaining support for both Debian/Ubuntu and RHEL/CentOS/Fedora systems

### Migration Order

1. **nginx-multisite cookbook** (moderate complexity, foundation for web services)
   - Start with basic Nginx installation and configuration
   - Add security hardening components
   - Implement SSL certificate generation
   - Configure multi-site setup

2. **cache cookbook** (low complexity, independent service)
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial cookbook** (high complexity, application deployment)
   - Set up PostgreSQL database
   - Deploy Python application with virtual environment
   - Configure systemd service

### Assumptions

1. The target environment will continue to support both Debian/Ubuntu and RHEL-based systems
2. Self-signed certificates are acceptable for development, but the Ansible solution should allow for easy integration of production certificates
3. The current security configurations are appropriate and should be maintained in the Ansible implementation
4. No changes to the application architecture are required as part of the migration
5. The Vagrant development environment will be maintained but converted to use Ansible provisioning
6. No external secret management system is required for the initial migration, but the solution should be designed to allow for future integration
7. The current Redis and PostgreSQL passwords will be migrated as-is but should be stored in Ansible Vault in the final implementation