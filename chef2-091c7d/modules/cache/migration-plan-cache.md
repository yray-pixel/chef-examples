---
source-path: cookbooks/cache
---

# Migration Plan: cache

**TLDR**: This cookbook configures two caching services: Memcached and Redis. It sets up a single Redis instance on port 6379 with password authentication and performs custom configuration modifications. The cookbook relies on external dependencies for the actual installation and configuration of both services.

## Service Type and Instances

**Service Type**: Cache

**Configured Instances**:

- **Redis 6379**: Primary Redis instance
  - Location/Path: /etc/redis/6379.conf
  - Port/Socket: 6379
  - Key Config: Password authentication enabled with 'redis_secure_password_123'
  - Log Directory: /var/log/redis

- **Memcached**: Default Memcached instance
  - Location/Path: Default from memcached cookbook
  - Port/Socket: Default (likely 11211)
  - Key Config: Default from memcached cookbook

## File Structure

```
cookbooks/cache/recipes/default.rb
cookbooks/cache/metadata.rb
```

## Module Explanation

The cookbook performs operations in this order:

1. **default** (`cookbooks/cache/recipes/default.rb`):
   - Includes the memcached recipe from the external memcached cookbook
   - Sets Redis configuration attributes:
     - Configures a single Redis instance on port 6379
     - Enables password authentication with 'redis_secure_password_123'
     - Disables replica-serve-stale-data setting
   - Creates Redis log directory at /var/log/redis
     - Owner: redis
     - Group: redis
     - Mode: 0755
     - Resources: directory (1)
   - Includes the redisio recipe from the external redisio cookbook
   - Executes a ruby_block to modify Redis configuration file
     - Removes several replication-related settings from /etc/redis/6379.conf:
       - replica-serve-stale-data
       - replica-read-only
       - repl-ping-replica-period
       - client-output-buffer-limit
       - replica-priority
     - Resources: ruby_block (1)
   - Includes the redisio::enable recipe to enable and start Redis service
   - Resources: include_recipe (3), directory (1), ruby_block (1)

## Dependencies

**External cookbook dependencies**:
- memcached (~> 6.0)
- redisio

**System package dependencies**:
- Redis server package (via redisio cookbook)
- Memcached package (via memcached cookbook)

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
- **Usage context**: Redis server authentication password used in Redis configuration

## Checks for the Migration

**Files to verify**:
- /etc/redis/6379.conf
- /var/log/redis (directory)
- Memcached configuration file (location depends on memcached cookbook)

**Service endpoints to check**:
- Ports listening:
  - Redis: 6379
  - Memcached: 11211 (default)
- Unix sockets: None specified
- Network interfaces: Default (likely 0.0.0.0 or 127.0.0.1)

**Templates rendered**:
None directly from this cookbook. Templates are likely rendered by the dependent cookbooks (memcached and redisio).

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
# These settings should be removed by the ruby_block

# Redis logs
tail -f /var/log/redis/redis_6379.log

# Redis directory permissions
ls -la /var/log/redis

# Memcached Service status
systemctl status memcached
ps aux | grep memcached

# Memcached connectivity
echo stats | nc localhost 11211
memcached-tool localhost:11211 stats

# Memcached configuration validation
cat /etc/memcached.conf

# Memcached logs
tail -f /var/log/memcached.log

# Network listening
netstat -tulpn | grep 6379  # Redis
ss -tlnp | grep redis
lsof -i :6379

netstat -tulpn | grep 11211  # Memcached
ss -tlnp | grep memcached
lsof -i :11211

# Memory usage
ps aux | grep redis | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS
ps aux | grep memcached | awk '{print $2}' | xargs -I {} cat /proc/{}/status | grep VmRSS

# Cache statistics
redis-cli -p 6379 -a 'redis_secure_password_123' info stats
echo stats | nc localhost 11211
```