# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backed by PostgreSQL. The migration to Ansible will involve converting 3 Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The codebase is well-structured with clear separation of concerns
- No custom resources or complex Chef-specific patterns are used
- Security configurations are straightforward and can be directly mapped to Ansible modules

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening (fail2ban, UFW firewall), and SSL certificate management
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening with fail2ban and UFW

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
- `solo.rb`: Chef Solo configuration - defines cookbook paths and logging settings
- `Vagrantfile`: Development environment configuration - uses Fedora 42 as the base OS with libvirt provider
- `vagrant-provision.sh`: Provisioning script for Vagrant - installs Chef and dependencies

### Target Details

Based on the source configuration files:

- **Operating System**: Fedora 42 (from Vagrantfile), with support for Ubuntu 18.04+ and CentOS 7+ (from cookbook metadata)
- **Virtual Machine Technology**: libvirt (from Vagrantfile)
- **Cloud Platform**: Not specified, appears to be targeting on-premises or generic VM deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` module and templates
- **memcached (~> 6.0)**: Replace with Ansible's `apt`/`yum` modules and templates for configuration
- **redisio (~> 7.2.4)**: Replace with Ansible's `apt`/`yum` modules and templates for Redis configuration

### Security Considerations

- **SSL Certificate Management**: The current implementation generates self-signed certificates. In Ansible, use the `openssl_*` modules for certificate generation or consider integrating with `community.crypto` collection.
- **Firewall Configuration**: Replace UFW commands with Ansible's `ufw` module or `firewalld` module depending on the target OS.
- **fail2ban Configuration**: Use Ansible templates to configure fail2ban similar to the current Chef implementation.
- **SSH Hardening**: Implement using Ansible's `lineinfile` module to modify SSH configuration.
- **Vault/secrets management**:
  - Redis password is hardcoded in the cache cookbook (`redis_secure_password_123`)
  - PostgreSQL credentials are hardcoded in the fastapi-tutorial cookbook (`fastapi`/`fastapi_password`)
  - Consider using Ansible Vault for these credentials

### Technical Challenges

- **Multi-site Nginx Configuration**: The current implementation dynamically generates site configurations based on node attributes. Implement similar logic using Ansible's template module and variables.
- **Service Dependencies**: Ensure proper ordering of service installations and configurations, particularly for the FastAPI application which depends on PostgreSQL.
- **SSL Certificate Generation**: Implement proper certificate management, potentially integrating with Let's Encrypt for production environments.
- **System Tuning**: The security configurations include sysctl parameters that need to be properly translated to Ansible.

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for web services)
   - Start with basic Nginx installation and configuration
   - Add SSL certificate management
   - Implement security hardening (fail2ban, firewall)
   - Configure virtual hosts

2. **cache** (low complexity, independent service)
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial** (high complexity, application deployment)
   - Set up PostgreSQL database
   - Deploy Python application from Git
   - Configure virtual environment and dependencies
   - Set up systemd service

### Assumptions

1. The target environment will continue to be Fedora/RHEL-based systems, with potential support for Ubuntu/Debian.
2. Self-signed certificates are acceptable for development, but production environments may require integration with a certificate authority.
3. The current security configurations are appropriate for the target environment and don't need significant changes.
4. The FastAPI application repository at `https://github.com/dibanez/fastapi_tutorial.git` will remain available and compatible.
5. The current Redis and Memcached configurations meet performance requirements and don't need optimization.
6. The current approach of deploying static HTML files for test sites will be maintained rather than implementing a more dynamic content management system.
7. The Vagrant development environment will be replaced with an equivalent Ansible-based local development solution.