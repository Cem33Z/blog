---
title: "SaltStack实践02-自定义master配置文件位置"
date: 2017-10-26 18:04:00
tags: [SaltStack]
---

## 环境说明
OS: CentOS-6.9-x86_64-minimal
SaltStack: Master/Minions version-2017.7.2

## Install
安装方法可以参考“SaltStack实践01-Master&Minions部署”

<!--more-->
## Config
1. salt-master 的主配置文件是`/etc/salt/master`
```
# 里面全都被注释了
[root@xxx-salt-master ~]# grep -Ev '^#|^$' /etc/salt/master | wc -l
0

# 虽然都被注释，但有些是默认配置项，意思是：你如果没指定配置，它就按默认的来，指定了就按你的来
```

2. `/etc/salt/master.d/` 目录里，只要是`*.conf` 结尾的配置文件，salt-master都会自动加载进去。所以我们把自定义的主配置文件都放在这个目录下，尽量做到不修改`/etc/salt/master`文件，以后管理、备份、迁移都会方便很多。

3. 另外，文章里讲到的是我常用的配置项，其他未列出配置项可以参考官方文档。[https://docs.saltstack.com/en/latest/ref/configuration/master.html](https://docs.saltstack.com/en/latest/ref/configuration/master.html)

### file_roots
这个配置项定义的目录，主要用来存放`.sls(salt state file) `文件。它可以定义不同的环境，比如测试环境、线上环境等，每一个环境可以有多个根目录，但是相同环境下多个根目录的子目录不能同名，否则 salt-master 不知道推送哪个文件给 minions 了。
```
# /etc/salt/master.d/file_roots.conf
file_roots:
  base:
    - /data/salt/base
```

###  pillar_roots
这个配置项定义的目录，主要用来存放全局静态数据，由它下发给minions（可以将一些涉密的敏感信息放这里面）。
```
# /etc/salt/master.d/pillar_roots.conf 
pillar_roots:
  base:
    - /data/salt/pillar
```

###  node-groups include
就是一个分组的功能，将不同用途的 HOST 按照你需求分组，用于执行 salt 命令时更好的匹配。我这里是按照业务分成单个不同的 conf 文件，你也可以写在同一个 conf 里。
```
# /etc/salt/master.d/node-groups_include.conf
include:
  - /data/salt/node-groups/*.conf
# /data/salt/node-groups/xxx.conf
nodegroups:
  xxx-web: 'xxx-web-01'
# /data/salt/node-groups/yyy.conf
nodegroups:
  yyy-db: 'yyy-db-01,yyy-db-02'
```

> [NODE GROUPS](https://docs.saltstack.com/en/latest/topics/targeting/nodegroups.html)
###  reactor
> event系统：Salt底层构建了Event BUS，所有操作均会产生Event。例如新的 minion 认证时，就会产生 tag 为 `salt/auth` 的 event，如启动 minion 时，就会产生tag为 `salt/minion/<hostname>/start`的 event

Reactor 就是匹配 event 系统的 tag，如果发现有和我定义相匹配的 tag，那么就可以触发我设定的 sls 来执行相应的操作。后续文章里，**新服务器安装了 salt-minion 后自动注册到 salt 系统里**，和 **salt-minion 服务启动后自动同步  master 上的配置**就是通过 Reactor 来实现的。
```
# /etc/salt/master.d/reactor.conf 
reactor:
  - 'salt/auth':
    - /data/salt/reactor/auth-pending.sls
  - 'salt/minion/txy-*/start':
    - /data/salt/reactor/auth-complete.sls
```

>[REACTOR SYSTEM](https://docs.saltstack.com/en/latest/topics/reactor/)
###  rosters include
这个配置项定义的文件，主要用于 salt-ssh 连接时需要的IP、ssh帐号密码这类信息。
```
# /etc/salt/master.d/rosters_include.conf 
roster_file: /data/salt/roster.d/rosters
# /data/salt/roster.d/rosters
xxx-web-02:
  host: 192.168.0.100
  user: root
  passwd: 123456
  port: 22
  timeout: 10 
```
## 参考资料
> [Salt (3) 核心所在：State](http://ohmystack.com/articles/salt-3-state)
> [salt-master配置文件详解](http://lansgg.blog.51cto.com/5675165/1541066)

