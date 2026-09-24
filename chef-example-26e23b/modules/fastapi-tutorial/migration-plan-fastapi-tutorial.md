---
source-path: cookbooks/fastapi-tutorial
---

# Migration Plan: fastapi-tutorial

**TLDR**: This cookbook deploys a FastAPI Python web application with a PostgreSQL database backend. It sets up a single FastAPI instance running on port 8000, configures a PostgreSQL database, and creates a systemd service to manage the application.

## Service Type and Instances

**Service Type**: Application Server (Python FastAPI web application)

**Configured Instances**:

- **fastapi-tutorial**: Python FastAPI web application
  - Location/Path: /opt/fastapi-tutorial
  - Port/Socket: 8000
  - Key Config: Uses PostgreSQL database, runs as systemd service

## File Structure

```
recipes/default.rb
metadata.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/fastapi-tutorial/recipes/default.rb`):
   - Installs Python 3 and required system packages: python3, python3-pip, python3-venv, git, postgresql, postgresql-contrib, libpq-dev
   - Creates application directory: /opt/fastapi-tutorial
   - Clones FastAPI tutorial repository from https://github.com/dibanez/fastapi_tutorial.git (branch: main)
   - Creates Python virtual environment at /opt/fastapi-tutorial/venv
   - Installs Python dependencies from requirements.txt
   - Configures PostgreSQL service (enables and starts)
   - Creates PostgreSQL database and user:
     - User: fastapi with password: fastapi_password
     - Database: fastapi_db
     - Grants all privileges on fastapi_db to fastapi user
   - Creates environment configuration file at /opt/fastapi-tutorial/.env with:
     - PROJECT_NAME="FastAPI Tutorial"
     - API_VERSION=1.0.0
     - DATABASE_URL=postgresql://fastapi:fastapi_password@localhost/fastapi_db
   - Creates systemd service file at /etc/systemd/system/fastapi-tutorial.service
   - Reloads systemd configuration when service file changes
   - Enables and starts the fastapi-tutorial service
   - Resources: package (1), directory (1), git (1), execute (3), service (2), file (2)

## Dependencies

**External cookbook dependencies**: None specified in metadata.rb
**System package dependencies**: python3, python3-pip, python3-venv, git, postgresql, postgresql-contrib, libpq-dev
**Service dependencies**: postgresql.service (required by fastapi-tutorial.service)

## Credentials

**Detection Summary**: 1 credential detected across 2 files

**Source**:
  - **Provider**: Hardcoded
  - **URL**: N/A
  - **Path**: N/A

### Database Password

- **Variable(s)**: 'fastapi_password'
- **Source file(s)**: cookbooks/fastapi-tutorial/recipes/default.rb
- **Current storage**: hardcoded
- **Usage context**: PostgreSQL database user password, used for database connection in the FastAPI application

## Checks for the Migration

**Files to verify**:
- /opt/fastapi-tutorial/.env
- /etc/systemd/system/fastapi-tutorial.service
- /opt/fastapi-tutorial/venv (Python virtual environment)
- /opt/fastapi-tutorial (application directory with cloned repository)

**Service endpoints to check**:
- Ports listening: 8000
- Network interfaces: 0.0.0.0 (listens on all interfaces)

**Templates rendered**:
- No templates used, but two files are created with inline content:
  - /opt/fastapi-tutorial/.env
  - /etc/systemd/system/fastapi-tutorial.service

## Pre-flight checks:
```bash
# Service status
systemctl status fastapi-tutorial
ps aux | grep uvicorn

# Application health
curl -I http://localhost:8000/
curl -s http://localhost:8000/docs  # FastAPI auto-generated docs should be available

# Database connectivity
sudo -u postgres psql -c "\l" | grep fastapi_db  # Verify database exists
sudo -u postgres psql -c "\du" | grep fastapi  # Verify user exists
PGPASSWORD=fastapi_password psql -h localhost -U fastapi -d fastapi_db -c "SELECT 1;"  # Test connection

# Configuration validation
cat /opt/fastapi-tutorial/.env | grep -E 'PROJECT_NAME|API_VERSION|DATABASE_URL'
cat /etc/systemd/system/fastapi-tutorial.service
systemctl show fastapi-tutorial | grep ExecStart

# Python environment
ls -la /opt/fastapi-tutorial/venv/bin/  # Check virtual environment
/opt/fastapi-tutorial/venv/bin/python --version  # Check Python version
/opt/fastapi-tutorial/venv/bin/pip list  # Check installed packages

# Logs
journalctl -u fastapi-tutorial -f  # Check service logs
tail -f /var/log/postgresql/postgresql-*.log  # Check PostgreSQL logs

# Network listening
netstat -tulpn | grep 8000
ss -tlnp | grep 8000
lsof -i :8000

# Repository check
ls -la /opt/fastapi-tutorial/
git -C /opt/fastapi-tutorial/ remote -v  # Verify git repository
git -C /opt/fastapi-tutorial/ branch  # Check current branch
```