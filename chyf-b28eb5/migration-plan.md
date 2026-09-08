# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site web application environment with caching services and a FastAPI application. The migration scope includes 3 Chef cookbooks with moderate complexity. Based on the analysis, we estimate a 2-3 week timeline for migration, with the most complex components being the multi-site Nginx configuration and security hardening.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled subdomains, security hardening, and firewall configuration
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, fail2ban integration, UFW firewall rules, sysctl security settings

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration, service management

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database configuration, systemd service management

### Infrastructure Files

- `Berksfile`: Chef dependency manager file listing cookbook dependencies and sources
- `solo.json`: Chef Solo configuration file with run list and node attributes
- `solo.rb`: Chef Solo configuration file
- `Vagrantfile`: Vagrant configuration for development environment
- `vagrant-provision.sh`: Shell script for Vagrant provisioning

### Target Details

Analyze the source repository to determine target environment specifications:

- **Operating System**: Ubuntu 18.04+ or CentOS 7+ (based on cookbook metadata support declarations)
- **Virtual Machine Technology**: Vagrant (based on Vagrantfile presence)
- **Cloud Platform**: Not specified in the examined files

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or community.general.nginx_* modules
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or package installation tasks

### Security Considerations

- **Firewall configuration**: Migrate UFW rules to Ansible's ufw module
- **fail2ban configuration**: Use Ansible to deploy fail2ban configuration files
- **SSH hardening**: Migrate SSH security settings using Ansible's lineinfile or template modules
- **sysctl security settings**: Use Ansible's sysctl module for kernel parameter configuration
- **Vault/secrets management**:
  - Hardcoded Redis password in cache cookbook
  - PostgreSQL database credentials in FastAPI cookbook
  - These should be migrated to Ansible Vault or an external secrets management solution

### Technical Challenges

- **Multi-site Nginx configuration**: The dynamic generation of multiple site configurations will require careful templating in Ansible
- **SSL certificate management**: Self-signed certificate generation logic needs to be replicated or replaced with Let's Encrypt integration
- **Custom resource usage**: The lineinfile custom resource will need to be replaced with Ansible's lineinfile module
- **Service dependencies**: Ensuring proper ordering of service installations and configurations, particularly for the FastAPI application which depends on PostgreSQL

### Migration Order

1. **cache cookbook** (low risk, standalone functionality)
2. **nginx-multisite cookbook** (moderate complexity, core infrastructure)
3. **fastapi-tutorial cookbook** (depends on properly configured web server)

### Assumptions

1. The current Chef setup is functional and represents the desired end state
2. No major architectural changes are planned during the migration
3. The target environment will continue to be Ubuntu/CentOS based systems
4. Self-signed certificates are acceptable for development, but production may require proper certificates
5. The FastAPI application repository at https://github.com/dibanez/fastapi_tutorial.git is accessible and contains the expected code
6. The current security configurations are appropriate for the target environment
7. No custom Chef handlers or complex search functionality is in use
8. The migration will maintain the same level of idempotence as the current Chef implementation