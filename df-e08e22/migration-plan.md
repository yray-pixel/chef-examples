# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site web application environment with caching services and a FastAPI application. The migration scope includes 3 Chef cookbooks with dependencies on external cookbooks from the Chef Supermarket. The complexity is moderate, with security configurations, SSL certificate management, and database integration requiring careful attention. Estimated timeline: 3-4 weeks for a complete migration, with an additional 1-2 weeks for testing and validation.

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx web server with multiple SSL-enabled subdomains, security hardening, and site configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, fail2ban integration, UFW firewall configuration, security hardening

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

- `Berksfile`: Dependency management file for Chef cookbooks, lists both local and external dependencies with version constraints
- `solo.json`: Chef Solo configuration file defining the run list and node attributes
- `solo.rb`: Chef Solo configuration settings
- `Vagrantfile`: Vagrant configuration for development/testing environment
- `vagrant-provision.sh`: Shell script for provisioning the Vagrant environment

### Target Details

Analyzing the source repository to determine target environment specifications:

- **Operating System**: Ubuntu 18.04+ or CentOS 7+ (based on metadata.rb support declarations)
- **Virtual Machine Technology**: Vagrant (based on Vagrantfile presence)
- **Cloud Platform**: Not specified in the examined files

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or nginx_core module
- **memcached (~> 6.0)**: Replace with Ansible memcached role or service module
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or service module

### Security Considerations

- **Firewall configuration**: UFW rules need to be migrated to Ansible ufw module
- **fail2ban integration**: Configuration needs to be migrated to Ansible fail2ban role
- **SSH hardening**: SSH configuration (disable root login, password authentication) needs to be migrated to Ansible ssh_config module
- **Vault/secrets management**: 
  - Redis password is hardcoded in the cache cookbook
  - PostgreSQL credentials are hardcoded in the fastapi-tutorial cookbook
  - SSL certificates are generated and managed in the nginx-multisite cookbook
  - Recommend using Ansible Vault for all credentials in the migrated solution

### Technical Challenges

- **SSL Certificate Management**: The current solution generates self-signed certificates. Consider using Ansible's openssl_* modules or integrating with Let's Encrypt via certbot role.
- **Multi-site Configuration**: The dynamic generation of Nginx site configurations based on node attributes will need careful translation to Ansible templates and variables.
- **Database Integration**: PostgreSQL database creation and user management will need to be handled by Ansible postgresql_* modules.
- **Service Dependencies**: Ensuring proper ordering of service installation, configuration, and startup in Ansible playbooks.

### Migration Order

1. **cache cookbook** (low complexity, standalone functionality)
2. **nginx-multisite cookbook** (moderate complexity, core infrastructure)
3. **fastapi-tutorial cookbook** (higher complexity, depends on database and potentially nginx)

### Assumptions

1. The target environment will continue to be Ubuntu 18.04+ or CentOS 7+ as specified in the cookbook metadata.
2. Self-signed certificates are acceptable for development, but production may require integration with a certificate authority.
3. The current security configurations (fail2ban, UFW, SSH hardening) are sufficient and should be maintained in the Ansible migration.
4. The FastAPI application source will continue to be pulled from the same Git repository.
5. The current Redis and PostgreSQL password strategies are acceptable, though they should be moved to Ansible Vault.
6. The Nginx site configuration structure will remain similar, with the same set of virtual hosts.
7. The current directory structure for document roots (/var/www/site.cluster.local) will be maintained.