---
title: "SaltStack实践08-系统参数修改sysctl.conf&ulimit&SElinux"
date: 2017-11.02 16:41:00
tags: [SaltStack]
---
## 环境说明
OS: CentOS-6.9-x86_64-minimal
SaltStack: Master/Minions version-2017.7.2

## Config salt-master
```
# /data/salt/base/init/system/init.sls 
include:
  - init.system.selinux
  - init.system.sysctl
  - init.system.ulimits
  - init.system.history
```
<!--more-->
### sysctl.conf
```
# /data/salt/base/init/system/sysctl.sls 
net.ipv4.conf.all.rp_filter:
  sysctl.present:
    - value: 1

net.ipv4.tcp_syncookies:
  sysctl.present:
    - value: 1

net.ipv4.ip_forward:
  sysctl.present:
    - value: 1

net.core.netdev_max_backlog:
  sysctl.present:
    - value: 655350

net.core.somaxconn:
  sysctl.present:
    - value: 655350

net.core.wmem_default:
  sysctl.present:
    - value: 8388608

net.core.rmem_default:
  sysctl.present:
    - value: 8388608

net.ipv4.tcp_mem:
  sysctl.present:
    - value: 78643200 104857600 157286400

net.ipv4.tcp_max_orphans:
  sysctl.present:
    - value: 655350

net.ipv4.tcp_fin_timeout:
  sysctl.present:
    - value: 30

net.ipv4.tcp_keepalive_time:
  sysctl.present:
    - value: 360

net.ipv4.icmp_ignore_bogus_error_responses:
  sysctl.present:
    - value: 1

net.ipv4.tcp_synack_retries:
  sysctl.present:
    - value: 2

net.ipv4.tcp_syn_retries:
  sysctl.present:
    - value: 2

net.ipv4.icmp_echo_ignore_all:
  sysctl.present:
    - value: 0

net.core.optmem_max:
  sysctl.present:
    - value: 409600

net.ipv4.ip_local_port_range:
  sysctl.present:
    - value: 1024 65000

net.core.rmem_max:
  sysctl.present:
    - value: 16777216

net.core.wmem_max:
  sysctl.present:
    - value: 16777216

net.ipv4.tcp_rmem:
  sysctl.present:
    - value: 4096 87380 16777216

net.ipv4.tcp_wmem:
  sysctl.present:
    - value: 4096 87380 16777216

net.ipv4.tcp_tw_recycle:
  sysctl.present:
    - value: 1

net.ipv4.tcp_tw_reuse:
  sysctl.present:
    - value: 1

net.ipv4.route.flush:
  sysctl.present:
    - value: 1

net.ipv4.tcp_max_syn_backlog:
  sysctl.present:
    - value: 180000

net.ipv4.tcp_max_tw_buckets:
  sysctl.present:
    - value: 8000

net.ipv4.tcp_ecn:
  sysctl.present:
    - value: 0

net.ipv4.neigh.default.gc_thresh3:
  sysctl.present:
    - value: 20480

net.ipv4.neigh.default.gc_thresh2:
  sysctl.present:
    - value: 10250

net.ipv4.neigh.default.gc_thresh1:
  sysctl.present:
    - value: 2560

net.ipv4.route.gc_thresh:
  sysctl.present:
    - value: 655350

vm.overcommit_memory:
  sysctl.present:
    - value: 1

kernel.printk_ratelimit_burst:
  sysctl.present:
    - value: 20

fs.file-max:
  sysctl.present:
    - value: 655350

net.ipv4.tcp_window_scaling:
  sysctl.present:
    - value: 1

net.ipv4.tcp_timestamps:
  sysctl.present:
    - value: 0

vm.swappiness:
  sysctl.present:
    - value: 0
```

### ulimits
```
# /data/salt/base/init/system/ulimits.sls 
/etc/security/limits.conf:
  file.managed:
    - source: salt://init/system/files/etc/security/limits.conf
    - user: root
    - group: root
    - mode: 644

  cmd.run:
    - name: ulimit -a
    - unless: grep 65535 /etc/security/limits.conf
```

### SElinux
```
# /data/salt/base/init/system/selinux.sls 
disabled_selinux:
  file.replace:
    - name: /etc/sysconfig/selinux
    - pattern: SELINUX=enforcing
    - repl: SELINUX=disabled

selinux_mode:
  selinux.mode:
    - name: permissive
```

### History
```
# /data/salt/base/init/system/history.sls 
/etc/profile:
  file.append:
    - name: /etc/profile
    - text: 'export HISTTIMEFORMAT="%F %T `whoami` "'
```

## Test
```
[root@xxx-salt-master ~]# salt 'xxx-salt-minion-01' state.sls init.system.sysctl
[root@xxx-salt-master ~]# salt 'xxx-salt-minion-01' state.sls init.system.selinux
[root@xxx-salt-master ~]# salt 'xxx-salt-minion-01' state.sls init.system.ulimits
[root@xxx-salt-master ~]# salt 'xxx-salt-minion-01' state.sls init.system.history
[root@xxx-salt-master ~]# salt 'xxx-salt-minion-01' state.sls init.system
```

## 参考资料

## 常见问题
### State 'selinux.mode' was not found in SLS 'init.system.selinux' Reason: 'selinux' __virtual__ returned False
```
[root@xxx-salt-master ~]# salt 'xxx-salt-minion-01' state.sls init.system.selinux
xxx-salt-minion-01:
----------
          ID: disabled_selinux
    Function: file.replace
        Name: /etc/sysconfig/selinux
      Result: True
     Comment: No changes needed to be made
     Started: 15:10:19.978797
    Duration: 5.073 ms
     Changes:   
----------
          ID: selinux_mode
    Function: selinux.mode
        Name: permissive
      Result: False
     Comment: State 'selinux.mode' was not found in SLS 'init.system.selinux'
              Reason: 'selinux' __virtual__ returned False
     Changes:   

Summary for xxx-salt-minion-01
------------
Succeeded: 1
Failed:    1
------------
Total states run:     2
Total run time:   5.073 ms
```

解决办法：
```
[root@xxx-salt-master ~]# salt 'xxx-salt-minion-02' cmd.run 'yum install -y policycoreutils-python selinux-policy-targeted'
```
