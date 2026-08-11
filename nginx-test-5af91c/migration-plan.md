# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx setup with FastAPI application and caching services. The migration to Ansible will involve converting 3 Chef cookbooks with their dependencies to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The repository has a clear structure with well-defined cookbooks
- No custom resources or complex Chef-specific patterns
- Standard infrastructure components (Nginx, FastAPI, PostgreSQL, Redis, Memcached)
- Security configurations that need careful migration

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Nginx web server with multiple SSL-enabled virtual hosts, security hardening, and self-signed certificate generation
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL/TLS setup, security hardening (fail2ban, ufw, sysctl)

- **fastapi-tutorial**:
    - Description: Python FastAPI application deployment with PostgreSQL database setup
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment, Git repository deployment, PostgreSQL database configuration, systemd service

- **cache**:
    - Description: Caching services configuration including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies - will be replaced by Ansible Galaxy requirements.yml
- `solo.json`: Contains Chef run list and configuration attributes - will be converted to Ansible group_vars
- `solo.rb`: Chef configuration file - not needed in Ansible
- `Vagrantfile`: VM configuration for testing - can be adapted for Ansible testing
- `vagrant-provision.sh`: Script to install Chef and run cookbooks - will be replaced by Ansible provisioning

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Libvirt (based on Vagrantfile provider configuration)
- **Cloud Platform**: Not specified, appears to be designed for on-premises deployment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible nginx role or nginx_core module
- **memcached (~> 6.0)**: Replace with Ansible memcached role or package installation tasks
- **redisio (~> 7.2.4)**: Replace with Ansible redis role or package installation tasks

### Security Considerations

- **SSL/TLS Certificate Management**: 
  - Self-signed certificates are generated for each site
  - Migration approach: Use Ansible's openssl_* modules for certificate generation
  - Consider integrating with Let's Encrypt for production environments

- **Firewall Configuration**: 
  - UFW is configured with default deny and specific allow rules
  - Migration approach: Use Ansible's ufw module to maintain identical configuration

- **Fail2ban Integration**: 
  - Fail2ban is configured for intrusion prevention
  - Migration approach: Create an Ansible role for fail2ban with equivalent configuration

- **SSH Hardening**: 
  - Root login disabled and password authentication disabled
  - Migration approach: Use Ansible's lineinfile or template module to configure SSH

- **Sysctl Security Settings**: 
  - Custom sysctl settings for security
  - Migration approach: Use Ansible's sysctl module to apply identical settings

- **Vault/secrets management**:
  - Redis password is hardcoded in the cache cookbook
  - PostgreSQL credentials are hardcoded in the fastapi-tutorial cookbook
  - Migration approach: Move all credentials to Ansible Vault

### Technical Challenges

- **Multi-site Nginx Configuration**: 
  - The current implementation uses templates to generate site configurations
  - Challenge: Maintaining the same flexibility in Ansible
  - Solution: Create a similar template structure in Ansible with site-specific variables

- **SSL Certificate Generation**: 
  - Self-signed certificates are generated for each site
  - Challenge: Ensuring certificates are only generated when needed
  - Solution: Use Ansible's openssl_* modules with proper changed_when conditions

- **Database Initialization**: 
  - PostgreSQL database and user creation with idempotent commands
  - Challenge: Ensuring idempotent database operations
  - Solution: Use Ansible's postgresql_* modules instead of raw SQL commands

- **Service Dependencies**: 
  - Proper ordering of service installation, configuration, and startup
  - Challenge: Maintaining correct dependency order
  - Solution: Use Ansible handlers and proper task ordering

### Migration Order

1. **nginx-multisite** (Priority 1)
   - Core infrastructure component that other services depend on
   - Start with basic Nginx installation and configuration
   - Add SSL/TLS certificate generation
   - Implement security hardening (fail2ban, ufw, sysctl)
   - Configure virtual hosts

2. **cache** (Priority 2)
   - Implement Memcached configuration
   - Implement Redis with authentication
   - Ensure proper service management

3. **fastapi-tutorial** (Priority 3)
   - Set up PostgreSQL database
   - Deploy Python application
   - Configure systemd service
   - Integrate with Nginx virtual host

### Assumptions

1. The target environment will continue to be Fedora-based systems (as indicated by the Vagrantfile)
2. Self-signed certificates are acceptable for development, but production may require proper CA-signed certificates
3. The security requirements (fail2ban, ufw, SSH hardening) will remain the same
4. The application deployment strategy (cloning from Git) will remain unchanged
5. The current directory structure in the target system (/opt/fastapi-tutorial, /opt/server/*) should be preserved
6. The current database credentials and Redis password are development values and should be replaced with proper secrets management in production
7. The Vagrant setup will be maintained for development and testing purposes