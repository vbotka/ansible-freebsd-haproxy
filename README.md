# freebsd_haproxy

[![license](https://img.shields.io/badge/license-BSD-red.svg)](https://www.freebsd.org/doc/en/articles/bsdl-gpl/article.html)
[![GitHub tag](https://img.shields.io/github/v/tag/vbotka/ansible-freebsd-haproxy)](https://github.com/vbotka/ansible-freebsd-haproxy/tags)

This role is included in the collection [vbotka.freebsd](https://galaxy.ansible.com/ui/repo/published/vbotka/freebsd/) as [vbotka.freebsd.haproxy](https://galaxy.ansible.com/ui/repo/published/vbotka/freebsd/content/role/haproxy/)

[Ansible role.](https://galaxy.ansible.com/vbotka/freebsd_haproxy/) FreeBSD. Install, configure, and run HAProxy.


## Requirements

### Collections

- community.general
- vbotka.freebsd


## Variables

Review defaults and examples in vars.


## Workflow

1) Change shell to /bin/sh

```bash
shell> ansible haproxy -e 'ansible_shell_type=csh ansible_shell_executable=/bin/csh' -a 'sudo pw usermod freebsd -s /bin/sh'
```

2) Install the role and collections

```bash
shell> ansible-galaxy role install vbotka.freebsd_haproxy
```

3) Install the collections

```bash
shell> ansible-galaxy collection install community.general
shell> ansible-galaxy collection install vbotka.freebsd
```

4) Change variables

- Review *defaults*
- Customize variables

5) Create the playbook *haproxy.yml*

```yaml
- hosts: haproxy
  roles:
    - vbotka.freebsd_haproxy
```

6) Create the inventory file *hosts*

```ini
[haproxy]
<haproxy-ip-or-fqdn>
[haproxy:vars]
ansible_connection=ssh
ansible_user=freebsd
ansible_become=yes
ansible_become_method=sudo
ansible_python_interpreter=auto_silent
```

7) Install and configure Haproxy

```bash
shell> ansible-playbook haproxy.yml
```

## References

- [HAProxy Documentation](http://haproxy.org/)


## License

[![license](https://img.shields.io/badge/license-BSD-red.svg)](https://www.freebsd.org/doc/en/articles/bsdl-gpl/article.html)


## Author Information

[Vladimir Botka](https://botka.info)


# Best Practices Guide

A guide to deploying, tuning, and securing [HAProxy](https://www.haproxy.org/)

## Table of Contents

- [1. Kernel & Network Tuning](#1-kernel--network-tuning)
- [2. Process Resource Limits](#2-process-resource-limits)
- [3. Optimized HAProxy Configuration](#3-optimized-haproxy-configuration)
- [4. Syslog Configuration](#4-syslog-configuration)
- [5. High Availability & Security Hardening](#5-high-availability-security-hardening)
  - [CARP Failover (Active / Passive)](#carp-failover-active-passive)
  - [Packet Filter (PF) Rate-Limiting](#pf-rate-limiting)

1. Kernel & Network Tuning

FreeBSD defaults are tuned for general-purpose server workloads. For
high-concurrency proxy traffic, append the following optimizations to
*/etc/sysctl.conf*:

```ini
# --- Open Files & Descriptors ---
kern.maxfiles=2097152
kern.maxfilesperproc=1048576

# --- Socket Backlog & Port Range ---
kern.ipc.somaxconn=65535
net.inet.ip.portrange.first=1024
net.inet.ip.portrange.last=65535

# --- Fast Recycling of Connections ---
net.inet.tcp.fast_finwait2_recycle=1
net.inet.tcp.finwait2_timeout=5000
net.inet.tcp.msl=7500

# --- Buffer Allocations & Memory Management ---
kern.ipc.nmbclusters=262144
kern.ipc.maxsockbuf=16777216

# --- TCP Buffer Auto-tuning ---
net.inet.tcp.sendbuf_max=16777216
net.inet.tcp.recvbuf_max=16777216
net.inet.tcp.sendbuf_auto=1
net.inet.tcp.recvbuf_auto=1

# --- Congestion Control ---
net.inet.tcp.cc.algorithm=cubic
```

Apply the changes immediately without rebooting:

```sh
sysctl -f /etc/sysctl.conf
```

2. Process Resource Limits

By default, the daemon class restricts file descriptors per process. Add a
dedicated profile for HAProxy in */etc/login.conf*:

```ini
haproxy:\
    :openfiles-cur=524288:\
    :openfiles-max=1048576:\
    :tc=daemon:
```

Recompile the login capabilities database:

```sh
cap_mkdb /etc/login.conf
```

3. Optimized HAProxy Configuration

Place this configuration in */usr/local/etc/haproxy.conf*:

```
global
    log /var/run/log local0 info
    chroot /var/empty
    user haproxy
    group haproxy
    daemon

    # CPU Affinity & Multi-threading
    nbthread 4
    cpu-map auto:1/1-4 0-3

    # Connection Limits
    maxconn 100000

    # Administrative Runtime Socket
    stats socket /var/run/haproxy.sock mode 600 level admin expose-fd listeners
    stats timeout 30s

    # Hardened TLS Settings
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    tune.ssl.cachesize 100000
    tune.ssl.lifetime 600

defaults
    log                     global
    mode                    http
    option                  httplog
    option                  dontlognull
    option                  http-keep-alive
    timeout connect         5000ms
    timeout client          50000ms
    timeout server          50000ms
    timeout http-request    10000ms
```

4. Syslog Configuration

Prevent HAProxy access logs from cluttering */var/log/messages*:

* Append this rule to */etc/syslog.conf*:

```
local0.*                                        /var/log/haproxy.log
```

* Configure log rotation in */etc/newsyslog.conf*:

```
/var/log/haproxy.log                    640  7     *    @T00  JC
```

5. High Availability & Security Hardening

**CARP Failover (Active / Passive)**

To run an active/passive HA cluster across two FreeBSD nodes using a shared Virtual IP (VIP):

* Allow CARP traffic and preempting in */etc/sysctl.conf*:

```ini
net.inet.carp.preempt=1
```

* Configure the shared VIP interface in */etc/rc.conf* (Node 1 - Master):

```ini
ifconfig_em0_alias0="vhid 1 advskew 0 pass MySecretKey 192.168.1.100/24"
```

* Configure the backup node (Node 2 - Backup):

```ini
ifconfig_em0_alias0="vhid 1 advskew 100 pass MySecretKey 192.168.1.100/24"
```

**Packet Filter (PF) Rate-Limiting**

Protect HAProxy from brute-force attacks and floods (*/etc/pf.conf*):

```
table <bruteforce> persist

block quick from <bruteforce>

pass in on $ext_if proto tcp to any port {80, 443} \
    flags S/SA keep state \
    (max-src-conn 200, max-src-conn-rate 100/5, overload <bruteforce> flush global)
```
