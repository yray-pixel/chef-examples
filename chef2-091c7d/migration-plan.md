# MIGRATION FROM CHEF TO ANSIBLE

## Executive Summary

This repository contains a Chef-based infrastructure configuration for a multi-site Nginx web server with caching services (Redis and Memcached) and a FastAPI application backend. The migration to Ansible will involve converting 3 Chef cookbooks with their recipes, templates, and attributes to equivalent Ansible roles and playbooks.

**Estimated Timeline:**
- Analysis and Planning: 1 week
- Development of Ansible roles: 2-3 weeks
- Testing and Validation: 1-2 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 5-7 weeks

**Complexity Assessment:** Medium
- The repository has a clear structure with well-defined cookbooks
- No custom resources or complex Chef-specific patterns
- Standard infrastructure components (Nginx, Redis, Memcached, PostgreSQL)
- Security configurations that need careful migration

## Module Migration Plan

This repository contains Chef cookbooks that need individual migration planning:

### MODULE INVENTORY

- **nginx-multisite**:
    - Description: Configures Nginx with multiple SSL-enabled virtual hosts, security hardening, and site configurations
    - Path: cookbooks/nginx-multisite
    - Technology: Chef
    - Key Features: Multi-site configuration, SSL certificate generation, security hardening (fail2ban, ufw), sysctl security settings

- **cache**:
    - Description: Configures caching services including Memcached and Redis with authentication
    - Path: cookbooks/cache
    - Technology: Chef
    - Key Features: Redis with password authentication, Memcached configuration

- **fastapi-tutorial**:
    - Description: Deploys a FastAPI Python application with PostgreSQL database backend
    - Path: cookbooks/fastapi-tutorial
    - Technology: Chef
    - Key Features: Python virtual environment setup, Git repository deployment, PostgreSQL database creation, systemd service configuration

### Infrastructure Files

- `Berksfile`: Defines cookbook dependencies (nginx, memcached, redisio)
- `solo.json`: Contains node attributes and run list for Chef Solo
- `solo.rb`: Chef Solo configuration file
- `vagrant-provision.sh`: Bash script for provisioning the Vagrant VM with Chef
- `Vagrantfile`: Defines the Vagrant VM configuration (Fedora 42)

### Target Details

- **Operating System**: Fedora 42 (based on Vagrantfile configuration)
- **Virtual Machine Technology**: Vagrant with libvirt provider
- **Cloud Platform**: Not specified, appears to be a local development environment

## Migration Approach

### Key Dependencies to Address

- **nginx (~> 12.0)**: Replace with Ansible's `nginx` role or direct package installation and configuration
- **memcached (~> 6.0)**: Replace with Ansible's `geerlingguy.memcached` role or direct package installation
- **redisio (~> 7.2.4)**: Replace with Ansible's `geerlingguy.redis` role or direct package installation and configuration

### Security Considerations

- **Firewall Configuration**: The Chef cookbook configures UFW. Migrate to Ansible's `firewalld` module for Fedora
- **Fail2ban Setup**: Migrate fail2ban configuration to Ansible tasks
- **SSH Hardening**: Preserve SSH security settings (disable root login, password authentication)
- **SSL Certificate Generation**: Migrate self-signed certificate generation to Ansible
- **Vault/secrets management**:
  - Redis password in cache cookbook: "redis_secure_password_123" (hardcoded)
  - PostgreSQL credentials in fastapi-tutorial cookbook: "fastapi_password" (hardcoded)
  - Consider migrating to Ansible Vault for secure credential storage

### Technical Challenges

- **Multi-site Nginx Configuration**: Ensure the dynamic generation of site configurations is preserved in Ansible
- **SSL Certificate Management**: Properly handle certificate generation and permissions
- **Service Dependencies**: Maintain proper ordering of service installations and configurations
- **Idempotency**: Ensure all operations remain idempotent, especially database user/schema creation

### Migration Order

1. **nginx-multisite** (moderate complexity, foundation for other services)
   - Start with basic Nginx installation
   - Add SSL certificate generation
   - Configure security components (fail2ban, firewall)
   - Implement multi-site configuration

2. **cache** (low complexity, independent service)
   - Implement Memcached configuration
   - Implement Redis with authentication

3. **fastapi-tutorial** (high complexity, depends on PostgreSQL)
   - Set up PostgreSQL
   - Configure Python environment
   - Deploy application from Git
   - Configure systemd service

### Assumptions

1. The target environment will continue to be Fedora-based systems
2. The same network configuration and port mappings will be maintained
3. Self-signed certificates are acceptable (no need for Let's Encrypt integration)
4. The FastAPI application repository will remain available at the same URL
5. No CI/CD pipeline integration is required for the initial migration
6. The Vagrant development environment will be preserved but updated to use Ansible provisioner

## Implementation Strategy

### 1. Directory Structure

Create an Ansible project with the following structure:

```
ansible-nginx-multisite/
├── inventory/
│   └── hosts.ini
├── group_vars/
│   └── all.yml
├── roles/
│   ├── nginx-multisite/
│   ├── cache/
│   └── fastapi-tutorial/
├── playbooks/
│   ├── site.yml
│   ├── nginx.yml
│   ├── cache.yml
│   └── fastapi.yml
└── Vagrantfile
```

### 2. Variable Migration

Convert Chef attributes from `solo.json` and cookbook attributes to Ansible variables in `group_vars/all.yml` and role defaults.

### 3. Template Migration

Migrate ERB templates to Jinja2 format, updating variable references accordingly.

### 4. Testing Strategy

1. Develop and test each role independently
2. Create integration tests for the complete environment
3. Validate against the original Chef-provisioned environment

## Knowledge Transfer Plan

1. Document each Ansible role with README files
2. Create a migration summary document highlighting key differences
3. Conduct a walkthrough session with the team
4. Provide examples of common maintenance tasks in the new Ansible environment