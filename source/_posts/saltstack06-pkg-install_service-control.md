---
title: "SaltStack实践06-软件包安装&服务起停"
date: 2017-11.02 16:02:00
tags: [SaltStack]
---

## 环境说明
OS: CentOS-6.9-x86_64-minimal
SaltStack: Master/Minions version-2017.7.2

## Config in salt-master
```
# /data/salt/base/init/pkg/init.sls
common_pkg_list:
  pkg.installed:
    - names: 
      - gcc
      - gcc-c++
      - man
      - wget
      - screen
      - zip
      - lsof
      - telnet
      - sysstat
      - tree
      # for control selinux in salt
      - policycoreutils-python
      - selinux-policy-targeted
```
<!--more-->
### Test
```
[root@xxx-salt-master ~]# salt 'xxx-salt-minion-01' state.sls init.pkg
```

## 参考资料

