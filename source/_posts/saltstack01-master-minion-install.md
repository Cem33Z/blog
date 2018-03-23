---
title: "SaltStack实践01-master&minion部署"
date: 2017-10-25 22:57:00
tags: [SaltStack]
---
## 环境说明
OS: CentOS-6.9-x86_64-minimal
SaltStack: Master/Minions version-2017.7.2

## Disable SElinux
```
# getenforce
Enforcing
# setenforce 0
# getenforce
Permissive
# sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
```
<!--more-->
## Update Hostname
```
1. in salt-master
[root@xxx-salt-master ~]# hostname_str=xxx-salt-master
[root@xxx-salt-master ~]# sed -i "s/^HOSTNAME=.*/HOSTNAME=${hostname_str}/g" /etc/sysconfig/network
[root@xxx-salt-master ~]# sed -i "$ a 127.0.0.1 ${hostname_str}" /etc/hosts
[root@xxx-salt-master-01 ~]# hostname ${hostname_str}
[root@xxx-salt-master-01 ~]# hostname -f

2. in salt-minion
[root@xxx-salt-minion-01 ~]# hostname_str=xxx-salt-minion-01
[root@xxx-salt-minion-01 ~]# sed -i "s/^HOSTNAME=.*/HOSTNAME=${hostname_str}/g" /etc/sysconfig/network
[root@xxx-salt-minion-01 ~]# sed -i "$ a 127.0.0.1 ${hostname_str}" /etc/hosts
[root@xxx-salt-minion-01 ~]# hostname ${hostname_str}
[root@xxx-salt-minion-01 ~]# hostname -f 
```

## Install
### Using the SaltStack repository
```
# yum install -y https://repo.saltstack.com/yum/redhat/salt-repo-latest-2.el6.noarch.rpm
# yum clean expire-cache

latest 为最新版本，如果需要安装指定版本，可以在 https://repo.saltstack.com/yum/redhat/ 里选择对应的 rpm 文件
```

### Install salt-master
```
[root@xxx-salt-master ~]# yum install salt-master -y
```

### Install salt-minion
```
[root@salt-minion ~]# yum install salt-minion -y
```

## Config Salt
### Start salt-master service
```
[root@xxx-salt-master ~]# chkconfig salt-master on
[root@xxx-salt-master ~]# service salt-master start
```

### Config and start salt-minion service
```
1. config salt-master ip
[root@xxx-salt-minion-01 ~]# sed -i "$ a master: 192.168.0.107" /etc/salt/minion
2. start salt-minion service
[root@xxx-salt-minion-01 ~]# chkconfig salt-minion on
[root@xxx-salt-minion-01 ~]# service salt-minion start
```

### Config salt-master
```
key-management:
[root@xxx-salt-master ~]# salt-key -L
Accepted Keys:
Denied Keys:
Unaccepted Keys:
salt-minion-01
Rejected Keys:
[root@xxx-salt-master ~]# salt-key -a xxx-salt-minion-01        # salt-key -A 允许所有
The following keys are going to be accepted:
Unaccepted Keys:
xxx-salt-minion-01
Proceed? [n/Y] y
Key for minion xxx-salt-minion-01 accepted.
Key for minion xxx-salt-minion-02 accepted.
[root@xxx-salt-master ~]# salt-key -L
Accepted Keys:
xxx-salt-minion-01
Denied Keys:
Unaccepted Keys:
Rejected Keys:

QA:
如果看不到 salt-minions 的 key，确认一下是否 salt-master 的防火墙没有允许 4505-4506 端口
[root@xxx-salt-master ~]# iptables -I INPUT -p tcp --dport 4505:4506 -j ACCEPT -m comment --comment "saltstack service"
[root@xxx-salt-master ~]# /etc/init.d/iptables save
```

## Test
```
Run in salt-master server
[root@xxx-salt-master ~]# salt '*' test.ping
xxx-salt-minion-01:
```
