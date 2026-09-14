---
source-path: cookbooks/cache
---

# Migration Plan: cache

**TLDR**: This cookbook configures two caching services: Memcached and Redis. It sets up Redis with authentication and custom configuration, including a specific port (6379) and password. The cookbook relies on external dependencies for the actual installation and configuration of both services.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **Redis**:
  - Location/Path: /etc/redis/6379.conf
  - Port/Socket: 6379
  - Key Config: Authentication enabled with password 'redis_secure_password_123'
  
- **Memcached**:
  - Location/Path: Default (determined by memcached cookbook)
  - Port/Socket: Default (typically 11211)
  - Key Config: Default configuration from memcached cookbook

## File Structure

```
cookbooks/cache/recipes/default.rb
cookbooks/cache/metadata.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **memcached** (external cookbook):
   - Includes the memcached cookbook recipe
   - Resources: include_recipe (1)

2. **redis configuration** (`cookbooks/cache/recipes/default.rb`):
   - Sets Redis configuration attributes:
     - Port: 6379
     - Password: 'redis_secure_password_123'
     - Disables replicaservestaledata
   - Creates Redis log directory at /var/log/redis
   - Resources: directory (1)

3. **redisio** (external cookbook):
   - Includes the redisio cookbook recipe
   - Resources: include_recipe (1)

4. **redis config fix** (`cookbooks/cache/recipes/default.rb`):
   - Executes a ruby_block to modify the Redis configuration file
   - Removes several replica-related configuration lines from /etc/redis/6379.conf
   - Resources: ruby_block (1)

5. **redisio::enable** (external cookbook):
   - Includes the redisio::enable recipe to enable and start the Redis service
   - Resources: include_recipe (1)

## Dependencies

**External cookbook dependencies**:
- memcached (~> 6.0)
- redisio

**System package dependencies**:
- Redis server (installed by redisio cookbook)
- Memcached server (installed by memcached cookbook)

**Service dependencies**:
- redis service (managed by redisio cookbook)
- memcached service (managed by memcached cookbook)

## Credentials

**Detection Summary**: 1 credential detected across 1 file

**Source**:
  - **Provider**: Hardcoded
  - **URL**: N/A
  - **Path**: N/A

### Redis Authentication Password

- **Variable(s)**: `node.default['redisio']['servers'][0]['requirepass']`
- **Source file(s)**: cookbooks/cache/recipes/default.rb
- **Current storage**: hardcoded
- **Usage context**: Redis server authentication password used to secure Redis instance

## Checks for the Migration

**Files to verify**:
- /etc/redis/6379.conf
- /var/log/redis/
- Memcached configuration files (location depends on memcached cookbook)

**Service endpoints to check**:
- Ports listening: 6379 (Redis), 11211 (Memcached default)
- Unix sockets: None specified
- Network interfaces: Default (0.0.0.0 or 127.0.0.1 depending on default configuration)

**Templates rendered**:
- None directly in this cookbook (templates would be rendered by the dependency cookbooks)

## Pre-flight checks:
```bash
# Redis Service status
systemctl status redis-server
systemctl status redis@6379
ps aux | grep redis

# Redis connectivity
redis-cli -p 6379 ping
redis-cli -p 6379 -a 'redis_secure_password_123' ping
redis-cli -p 6379 -a 'redis_secure_password_123' info server

# Redis configuration validation
cat /etc/redis/6379.conf | grep -E 'port|requirepass'
cat /etc/redis/6379.conf | grep -E 'replica-serve-stale-data|replica-read-only|repl-ping-replica-period|client-output-buffer-limit|replica-priority'

# Redis logs
tail -f /var/log/redis/redis_6379.log

# Redis network listening
netstat -tulpn | grep 6379
ss -tlnp | grep redis
lsof -i :6379

# Redis data directories
ls -lah /var/lib/redis/
ls -lah /var/log/redis/

# Memcached Service status
systemctl status memcached
ps aux | grep memcached

# Memcached connectivity
echo stats | nc localhost 11211
memcached-tool localhost:11211 stats

# Memcached configuration validation
cat /etc/memcached.conf

# Memcached logs
journalctl -u memcached -f

# Memcached network listening
netstat -tulpn | grep 11211
ss -tlnp | grep memcached
lsof -i :11211

# Memory usage
free -m
ps aux | grep redis | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS
ps aux | grep memcached | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS
```