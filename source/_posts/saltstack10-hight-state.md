---
title: "SaltStack实践10-hight.state自动同步"
date: 2017-11.03 09:52:00
tags: [SaltStack]
---

## 环境说明
OS: CentOS-6.9-x86_64-minimal
SaltStack: Master/Minions version-2017.7.2

<!--more-->
## Config salt-master
```
# /data/salt/base/top.sls 
base:
  "*":
    - init
    - users

# /data/salt/base/
├── init
│   ├── init.sls
│   ├── iptables
│   │   └── init.sls
│   ├── pkg
│   │   └── init.sls
│   ├── screen
│   │   └── init.sls
│   ├── sshd
│   │   ├── files
│   │   │   └── etc
│   │   │       └── ssh
│   │   │           └── sshd_config
│   │   └── init.sls
│   └── system
│       ├── files
│       │   └── etc
│       │       └── security
│       │           └── limits.conf
│       ├── history.sls
│       ├── init.sls
│       ├── selinux.sls
│       ├── sysctl.sls
│       └── ulimits.sls
├── top.sls
└── users
    ├── add_manager.sls
    ├── add_sudoers.sls
    ├── chg_passwd.sls
    ├── files
    │   └── etc
    │       └── sudoers.d
    │           └── sudoers
    ├── init.sls
    └── README
```

## 测试手动同步
```
[root@xxx-salt-master ~]# salt '*' state.highstate
```

## 定期自动同步
```
# /data/salt/pillar/
├── README
├── schedule
│   └── init.sls
├── screen
│   └── init.sls
├── sshd
│   └── init.sls
├── top.sls
└── users
    └── init.sls

# /data/salt/pillar/top.sls 
base:
    "*":
      - users
      - sshd
      - screen
      - schedule

# /data/salt/pillar/schedule/init.sls 
schedule:
  highstate:
    function: state.highstate
    minutes: 2
```

### 测试自动同步
```
# 查看 sudoer 文件有没有增加用户
[root@xxx-salt-minion-01 ~]# watch -n 1 ls -l /etc/sudoers.d/
```

## 参考资料

