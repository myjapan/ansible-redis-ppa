# ansible-redis-ppa

 - Based on the awesome [ansible-redis](https://github.com/DavidWittman/ansible-redis) and [ansible-redis-ppa](https://github.com/davidcaste/ansible-redis-ppa)
 - Requires Ansible 1.6.3+.
 - Compatible with CentOS

## Contents

 1. [Motivation](#motivation)
 1. [Installation](#installation)
 1. [Getting Started](#getting-started)
  1. [Single Redis node](#single-redis-node)
  1. [Master-Slave Replication](#master-slave-replication)
  1. [Redis Sentinel](#redis-sentinel)
 1. [Installing redis from a source file in the ansible role](#installing-redis-from-a-source-file-in-the-ansible-role)
 1. [Configurables](#configurables)

## Motivation

Although the original [ansible-redis](https://github.com/DavidWittman/ansible-redis) role works very well it installs Redis from its sources. It's reasonable in some environments, but in a production system it isn't recommendable because a compiler and other tools are installed too.

The [ansible-redis-ppa](https://github.com/davidcaste/) is only for Debian-based distros, so this version is made to be CentOS compatible.

Redis is installed using [Remi Repo](https://blog.remirepo.net/)

## Getting started

Below are a few example playbooks and configurations for deploying a variety of Redis architectures.

This role expects to be run as root or as a user with sudo privileges.

### Single Redis node

Deploying a single Redis server node is pretty trivial; just add the role to your playbook and go.

``` yml
---
- hosts: redis-servers
  roles:
    - redis
```
inventory.yml:
``` yml
  redis-servers:
    rediscl01_master:
      hosts:
        redis0.example.com:
      vars: 
        install_redis_sentinel: True
        redis_sentinel_monitors:
          - name: redis0.example.com
            host: <master.server.ip.address>
            port: 6379

```

``` bash
$ ansible-playbook -i inventory.yml redis.yml
```

### Master-Slave replication

Configuring [replication](http://redis.io/topics/replication) in Redis is accomplished by deploying multiple nodes, and setting the `redis_slaveof` variable on the slave nodes, just as you would in the redis.conf. In the example that follows, we'll deploy a Redis master with three slaves.

In this example, we're going to use groups to separate the master and slave nodes. Let's start with the inventory file:

``` yml
  redis-servers:
    rediscl01_master:
      hosts:
        redis-master.example.com:
    rediscl01_slave:
      hosts:
        redis-slave.example.com:
      vars:
        redis_slaveof: redis-master.example.com 6379
```

And here's the playbook:

``` yml
---
- hosts: redis-servers
  roles:
    - redis
```

In this case, I'm assuming you have DNS records set up for redis-master.example.com, but that's not always the case. You can pretty much go crazy with whatever you need this to be set to. In many cases, I tell Ansible to use the eth1 IP address for the master. Here's a more flexible value for the sake of posterity:

``` yml
redis_slaveof: "{{ hostvars['redis-master.example.com'].ansible_eth1.ipv4.address }} {{ redis_port }}"
```


### Redis Sentinel

#### Introduction

Using Master-Slave replication is great for durability and distributing reads and writes, but not so much for high availability. If the master node fails, a slave must be manually promoted to master, and connections will need to be redirected to the new master. The solution for this problem is [Redis Sentinel](http://redis.io/topics/sentinel), a distributed system which uses Redis itself to communicate and handle automatic failover in a Redis cluster.

Sentinel itself uses the same redis-server binary that Redis uses, but runs with the `--sentinel` flag and with a different configuration file. All of this, of course, is abstracted with this Ansible role, but it's still good to know.

#### Configuration

To add a Sentinel node to an existing deployment, assign this same `redis` role to it, and set the variable `install_redis_sentinel` to True on that particular host. This can be done in any number of ways, and for the purposes of this example I'll extend on the inventory file used above in the Master/Slave configuration:

``` yml
  redis-servers:
    rediscl01_master:
      hosts:
        redis-master.example.com:
      vars: 
        install_redis_sentinel: True
        redis_sentinel_monitors:
          - name: redis-master.example.com
            host: <master.server.ip.address>
            port: 6379
    rediscl01_slave:
      hosts:
        redis-slave.example.com:
      vars:
        redis_slaveof: redis-master.example.com 6379
        install_redis_sentinel: True
        redis_sentinel_monitors:
          - name: redis-master.example.com
            host: <master.server.ip.address>
            port: 6379
    rediscl01_sentinel:
      hosts:
        redis-sentinel.example.com:
      vars:
        install_redis_server: False
        install_redis_sentinel: True
        redis_sentinel_monitors:
          - name: redis-master.example.com
            host: <master.server.ip.address>
            port: 6379      

```

Above, we've set the `install_redis_sentinel` variable inline within the inventory file.

And, again, here's the playbook:

``` yml
---
- hosts: redis-servers
  roles:
    - redis
```

This will configure the Sentinel nodes to monitor the master we created above using the identifier `redis-master.example.com`. By default, Sentinel will use a quorum of 2, which means that at least 2 Sentinels must agree that a master is down in order for a failover to take place. This value can be overridden by setting the `quorum` key within your monitor definition. See the [Sentinel docs](http://redis.io/topics/sentinel) for more details.

Along with the variables listed above, Sentinel has a number of its own configurables just as Redis server does. These are prefixed with `redis_sentinel_`, and are enumerated in the **Configurables** section below.

Note: the third server has only the sentinel installed --- but in CentOS redis-sentinel is not a separate package from redis, so we need to use the variable `install_redis_server: False` in order to disable redis server and keep redis-sentinel active.


## Configurables

Here is a list of all the default variables for this role, which are also available in defaults/main.yml. 

``` yml
---
## General installation options

## Redis Server options
install_redis_server: true
redis_daemonize: "yes"
redis_dir: /var/lib/redis
redis_pidfile: /var/run/redis/redis-server.pid
redis_databases: 16
redis_protected_mode: "no"

# Networking/connection options
redis_bind: 0.0.0.0
redis_port: 6379
redis_password:  false
redis_tcp_backlog: 511
redis_timeout: 0
redis_tcp_keepalive: 0
redis_maxclients: 10000

# Socket options
# Set socket_path to the desired path to the socket. E.g. /var/run/redis/redis-server.sock
redis_socket_path: false
redis_socket_perm: 755

# Logging
redis_logfile: /var/log/redis/redis.log
redis_loglevel: notice
redis_syslog_enabled: "no"
redis_syslog_ident: redis
redis_syslog_facility: local0

# Replication
# Set slaveof just as you would in redis.conf. (e.g. "redis01 6379")
redis_slaveof: false
redis_slave_priority: 100
redis_slave_serve_stale_data: "yes"
redis_slave_read_only: "yes"
redis_repl_diskless_sync: "no"
redis_repl_diskless_sync_delay: 5
redis_repl_ping_slave_period: false
redis_repl_timeout: false
redis_repl_disable_tcp_nodelay: "no"
redis_repl_backlog_size: false
redis_repl_backlog_ttl: false

# Security
redis_rename_commands: []

# Snapshotting
redis_save:
  - 900 1
  - 300 10
  - 60 10000
redis_stop_writes_on_bgsave_error: "yes"
redis_rdbcompression: "yes"
redis_rdbchecksum: "yes"
redis_dbfilename: dump.rdb

# Limits
redis_maxmemory: false
redis_maxmemory_policy: noeviction
redis_appendonly: "no"
redis_appendfilename: appendonly.aof
redis_appendfsync: everysec
redis_no_appendfsync_on_rewrite: "no"
redis_auto_aof_rewrite_percentage: 100
redis_auto_aof_rewrite_min_size: 64mb
redis_aof_load_truncated: "yes"

redis_slowlog_log_slower_than: 10000
redis_slowlog_max_len: 128

redis_latency_monitor_threshold: 0

redis_nofile_limit: 16384
#redis_disable_thp: false

redis_sysctl_tweaks:
  - name: "net.core.somaxconn"
    value: 16384
    state: present
  - name: "vm.overcommit_memory"
    value: 1
    state: present


## Redis Sentinel options
install_redis_sentinel: false

redis_sentinel_bind: 0.0.0.0
redis_sentinel_port: 26379

redis_sentinel_dir: /var/lib/redis
redis_sentinel_pidfile: /var/run/redis/redis-sentinel.pid
redis_sentinel_logfile: /var/log/redis/sentinel.log
redis_sentinel_syslog_ident: sentinel

sentinel_sysctl_tweaks:
  - name: "net.core.somaxconn"
    value: 16384
    state: present

redis_sentinel_monitors:
  - name: master01
    host: localhost
    port: 6379
#    quorum: 2
#    auth_pass: ant1r3z
#    down_after_milliseconds: 30000
#    parallel_syncs: 1
#    failover_timeout: 180000
#    notification_script: false
#    client_reconfig_script: false
```

## Facts

The following facts are accessible in your inventory or tasks outside of this role.

- `{{ ansible_local.redis.bind }}`
- `{{ ansible_local.redis.port }}`
- `{{ ansible_local.redis.sentinel_bind }}`
- `{{ ansible_local.redis.sentinel_port }}`
- `{{ ansible_local.redis.sentinel_monitors }}`
