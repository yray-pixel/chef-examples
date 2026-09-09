---
source-path: cookbooks/cache
---

# Migration Plan: cache

**TLDR**: This cookbook configures two caching services: Memcached and Redis. It sets up Redis with authentication and custom configuration, including a log directory. The cookbook relies on external dependencies for both Memcached and Redis core functionality.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **Redis**:
  - Location/Path: /etc/redis/6379.conf
  - Port/Socket: 6379
  - Key Config: Authentication enabled with password, custom log directory at /var/log/redis

- **Memcached**:
  - Location/Path: Not specified in this cookbook (handled by dependency)
  - Port/Socket: 11211
  - Key Config: Not specified in this cookbook (handled by dependency)

## File Structure

```
cookbooks/cache/recipes/default.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/cache/recipes/default.rb`):
   - Includes the memcached recipe from external dependency
   - Sets Redis configuration attributes:
     - Port: 6379
     - Password: redis_secure_password_123
     - Disables replicaservestaledata
   - Creates Redis log directory at /var/log/redis
     - Owner: redis
     - Group: redis
     - Mode: 0755
     - Recursive: true
   - Includes the redisio recipe from external dependency
   - Executes a ruby_block to modify Redis configuration:
     - Removes specific replication and client buffer settings from /etc/redis/6379.conf
     - Specifically removes:
       - replica-serve-stale-data
       - replica-read-only
       - repl-ping-replica-period
       - client-output-buffer-limit
       - replica-priority
   - Includes the redisio::enable recipe from external dependency
   - Resources: include_recipe (3), directory (1), ruby_block (1)

## Dependencies

**External cookbook dependencies**:
- memcached (~> 6.0)
- redisio

**System package dependencies**:
- Redis server (installed by redisio dependency)
- Memcached server (installed by memcached dependency)

**Service dependencies**:
- redis service (managed by redisio dependency)
- memcached service (managed by memcached dependency)

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
- **Usage context**: Redis server authentication password, used to secure Redis instance

## Checks for the Migration

**Files to verify**:
- /etc/redis/6379.conf
- /var/log/redis/

**Service endpoints to check**:
- Ports listening: 6379 (Redis), 11211 (Memcached)
- Unix sockets: None specified
- Network interfaces: Not specified (likely default to all interfaces)

**Templates rendered**:
- None directly from this cookbook (handled by dependencies)

## Pre-flight checks:
```bash
# Redis Service status
systemctl status redis-server
systemctl status redis@6379
ps aux | grep redis

# Redis connectivity with authentication
redis-cli -p 6379 ping
redis-cli -p 6379 -a 'redis_secure_password_123' ping
redis-cli -p 6379 -a 'redis_secure_password_123' info server

# Redis configuration validation
cat /etc/redis/6379.conf | grep -E 'port|requirepass'
cat /etc/redis/6379.conf | grep -E 'replica-serve-stale-data|replica-read-only|repl-ping-replica-period|client-output-buffer-limit|replica-priority'

# Redis logs
ls -la /var/log/redis/
tail -f /var/log/redis/redis_6379.log

# Redis network listening
netstat -tulpn | grep 6379
ss -tlnp | grep redis
lsof -i :6379

# Memcached Service status
systemctl status memcached
ps aux | grep memcached

# Memcached connectivity
echo stats | nc localhost 11211
memcached-tool localhost:11211 stats

# Memcached network listening
netstat -tulpn | grep 11211
ss -tlnp | grep memcached
lsof -i :11211

# Directory permissions
ls -la /var/log/redis/
```