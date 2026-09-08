---
source-path: cookbooks/cache
---

# Migration Plan: cache

**TLDR**: This cookbook configures two caching services: Memcached and Redis. It sets up Redis with authentication and performs some configuration file modifications. The cookbook relies heavily on external dependencies for the actual installation and configuration of both services.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **Redis**:
  - Location/Path: /etc/redis/6379.conf
  - Port/Socket: 6379
  - Key Config: Authentication enabled with password 'redis_secure_password_123'
  
- **Memcached**:
  - Location/Path: Not explicitly defined in this cookbook (handled by dependency)
  - Port/Socket: Not explicitly defined in this cookbook (handled by dependency)
  - Key Config: Not explicitly defined in this cookbook (handled by dependency)

## File Structure

```
cookbooks/cache/recipes/default.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/cache/recipes/default.rb`):
   - Includes the memcached recipe from external cookbook
   - Sets Redis configuration attributes:
     - port: 6379
     - requirepass: redis_secure_password_123
     - replicaservestaledata: nil
   - Creates Redis log directory at /var/log/redis
     - Owner: redis
     - Group: redis
     - Mode: 0755
     - Resources: directory (1)
   - Includes the redisio recipe from external cookbook
   - Executes a ruby_block to modify Redis configuration file
     - Removes several replication-related configuration lines from /etc/redis/6379.conf
     - Resources: ruby_block (1)
   - Includes the redisio::enable recipe from external cookbook
   - Resources: include_recipe (3), directory (1), ruby_block (1)

## Dependencies

**External cookbook dependencies**:
- memcached (~> 6.0)
- redisio

**System package dependencies**:
- Redis (installed by redisio cookbook)
- Memcached (installed by memcached cookbook)

**Service dependencies**:
- redis service (managed by redisio cookbook)
- memcached service (managed by memcached cookbook)

## Credentials

**Detection Summary**: 1 credential detected in 1 file

**Source**:
  - **Provider**: Hardcoded
  - **URL**: N/A
  - **Path**: N/A

### Redis Authentication Password

- **Variable(s)**: `node.default['redisio']['servers'][0]['requirepass']`
- **Source file(s)**: cookbooks/cache/recipes/default.rb
- **Current storage**: hardcoded
- **Usage context**: Redis server authentication password

## Checks for the Migration

**Files to verify**:
- /etc/redis/6379.conf
- /var/log/redis/

**Service endpoints to check**:
- Ports listening: 6379 (Redis)
- Memcached port (typically 11211, but not explicitly defined in this cookbook)

**Templates rendered**:
No templates are directly rendered by this cookbook. Templates are likely rendered by the dependency cookbooks.

## Pre-flight checks:
```bash
# Redis Service status
systemctl status redis-server
systemctl status redis@6379
ps aux | grep redis

# Redis connectivity
redis-cli -h localhost -p 6379 ping
redis-cli -h localhost -p 6379 -a 'redis_secure_password_123' ping
redis-cli -h localhost -p 6379 -a 'redis_secure_password_123' info server

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
echo "stats" | nc localhost 11211
memcached-tool localhost:11211 stats

# Memcached configuration validation
cat /etc/memcached.conf

# Memcached logs
tail -f /var/log/memcached.log
journalctl -u memcached -f

# Memcached network listening
netstat -tulpn | grep memcached
ss -tlnp | grep memcached
lsof -i :11211
```