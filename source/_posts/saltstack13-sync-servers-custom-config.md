---
title: "SaltStack实践13-不同服务器之间的自定义配置同步"
date: 2017-11.03 17:03:00
tags: [SaltStack]
---

## 环境说明 ##
OS: CentOS-6.9-x86_64-minimal

SELinux: disabled
SaltStack: Master/Minions version-2017.7.2
salt-master: txy-salt-master 192.168.0.55
salt-minion:
txy-wzry-gamesvr-01 192.168.0.52
txy-wzry-websvr-02 192.168.0.53
txy-wzry-websvr-03 192.168.0.54

## 需求描述 ##
有几个不同用途服务器集群，需要使用 salt 统一管理：
* 不同点：每台服务器 iptables 里开放端口不一样；
  gameserver 服务器组对外端口是 8000，webserver 服务器组对外端口是 8080
* 相同点：比如SSH服务在iptable里的允许连接的IP、创建的登录用户；
  所有服务器都需要创建 xdx 用户，以及对内网开放 SSH 端口

<!--more-->
### 处理思路 ###
1.通过 nodegroup 对不同用途的服务器分组
2.在 pillar 里配置：相同的地方使用基础 sls 文件，不同的地方根据分组对应不同的 sls 文件
3.基础 sls 文件里编写所有服务器需要创建的用户，和对内网开放 SSH 端口；
4.在自定义的 sls 文件里编写不同功能的服务器 iptables 里开放端口
## install ##
参考 [SaltStack实践01-master&minion部署](http://www.off.gg/2017/10/25/saltstack01-master-minion-install/)

## configure ##
### salt-master　配置　###
1.主配置文件
```
# tree /etc/salt/master.d/
/etc/salt/master.d/
├── file_roots.conf
├── node-groups_include.conf
└── pillar_roots.conf


# /etc/salt/master.d/file_roots.conf
file_roots:
  base:
    - /data/salt/base

# /etc/salt/master.d/node-groups_include.conf
include:
  - /data/salt/node-groups/*.conf

# /etc/salt/master.d/pillar_roots.conf
pillar_roots:
  base:
    - /data/salt/pillar/base
  prod:
    - /data/salt/pillar/netgame
```
2.node-groups 配置
```
# tree /data/salt/node-groups/
/data/salt/node-groups/
└── wzry.conf

# /data/salt/node-groups/wzry.conf
nodegroups:
  wzry.prod.gameserver: 'E@txy-wzry-gamesvr-*'
  wzry.prod.webserver: 'E@txy-wzry-websvr-*'
  wzry: 'N@wzry.prod.gameserver or N@wzry.prod.webserver'
```
3.file_roots 配置
```
# tree /data/salt/base
/data/salt/base
├── iptables.sls
├── top.sls
└── users.sls

# /data/salt/base/top.sls 
base:
  "*":
    - iptables
    - users

# /data/salt/iptables.sls
{% for eachfw, fw_rule in pillar['firewall'].iteritems() %}
# Add custom chain
{{ eachfw }}-chain:
  iptables.chain_present:
    - name: {{ eachfw }}-chain
    - family: ipv4

# Custom chain rules
{% if 'allow' in fw_rule %}
# White Lists
{% for each_allow in fw_rule['allow'] %}
{{ eachfw }}_allow_{{ each_allow }}:
  iptables.insert:
    - table: filter
    - chain: {{ eachfw }}-chain
    - position: 1
    - source: {{ each_allow }}
    - jump: ACCEPT
    - require:
      - iptables: {{ eachfw }}-chain
    - require_in:
      - iptables: {{ eachfw }}_deny
    - save: True
{% endfor %}
# Deny all
{{ eachfw }}_deny:
  iptables.append:
    - table: filter
    - chain: {{ eachfw }}-chain
    - jump: DROP
    - save: True

{% elif 'deny' in fw_rule %}
# Black Lists
{% for each_deny in fw_rule['deny'] %}
{{ eachfw }}_deny_{{ each_deny }}:
  iptables.insert:
    - table: filter
    - chain: {{ eachfw }}-chain
    - position: 1
    - source: {{ each_deny }}
    - jump: DROP
    - require:
      - iptables: {{ eachfw }}-chain
    - require_in:
      - iptables: {{ eachfw }}_allow
    - save: True
{% endfor %}
# Accept all
{{ eachfw }}_allow:
  iptables.append:
    - table: filter
    - chain: {{ eachfw }}-chain
    - jump: ACCEPT
    - save: True
{% endif %}

# Export traffic to custom chain
{{ eachfw }}-main:
  iptables.insert:
    - table: filter
    - chain: INPUT
    - position: 1
    - proto: tcp
    - dport: {{ fw_rule['port'] }}
    - jump: {{ eachfw }}-chain
    - save: True
{% endfor %}

# /data/salt/base/users.sls 
{% set login_users = pillar.get('login_user_list', {}).items() %}
{% for user, args in login_users %}
{{ user }}:
  user.present:
    - shell: {{ args['shell'] }}
    - password: {{ args['passwd'] }}
{% endfor %}
```
3.pillar_roots 配置
```
# tree /data/salt/pillar
/data/salt/pillar
├── base
│   ├── sshd.sls
│   ├── top.sls
│   └── users.sls
└── netgame
    └── wzry
        ├── gamesvr.sls
        ├── init.sls
        └── websvr.sls

# /data/salt/pillar/base/top.sls 
base:
    "*":
      - sshd
      - users
prod:
  wzry:
    - match: nodegroup
    - wzry

# /data/salt/pillar/base/sshd.init.sls 
firewall:
  sshd_firewall:
    port: 22
    allow:
      - 192.168.0.0/24

# /data/salt/pillar/base/users.sls 
login_user_list:
  xdx:
  	passwd: '$1$root$zK.nSfat5qa/vIFd7jttn/'
    shell: /bin/bash

# /data/salt/pillar/netgame/wzry/init.sls 
include:
  - wzry.users
{% if grains['id'].startswith('txy-wzry-gamesvr-') %}
  - wzry.gamesvr
{% elif grains['id'].startswith('txy-wzry-websvr-') %}
  - wzry.websvr
{% endif %}

# /data/salt/pillar/netgame/wzry/gamesvr.sls 
firewall:
  wzry.gamesvr:
    port: 8000
    allow:
      - 192.168.0.0/24

# /data/salt/pillar/netgame/wzry/websvr.sls 
firewall:
  wzry.websvr:
    port: 8080
    allow:
      - 0.0.0.0/0
```

## Test ##
```
# salt '*' pillar.get firewall
txy-wzry-websvr-02:
    ----------
    wzry.websvr:
        ----------
        allow:
            - 0.0.0.0/0
        port:
            8080
    sshd_firewall:
        ----------
        allow:
            - 192.168.0.0/24
        port:
            22
txy-wzry-gamesvr-01:
    ----------
    wzry.gamesvr:
        ----------
        allow:
            - 192.168.0.0/24
        port:
            8000
    sshd_firewall:
        ----------
        allow:
            - 192.168.0.0/24
        port:
            22
txy-wzry-websvr-03:
    ----------
    wzry.websvr:
        ----------
        allow:
            - 0.0.0.0/0
        port:
            8080
    sshd_firewall:
        ----------
        allow:
            - 192.168.0.0/24
        port:
            22
```

## 总结 ##
如何利用 saltstack 统一管理不同服务器，或者不同业务服务器之间的的自定义配置问题，当时也是测试了好几个方案（例如 grains,mine），后来发现 nodegroup+pillar 方式比较简单。
* mine 适用与不同 minions 之间共享信息；
* grains 有两种方式：如果由 salt-master 下发给 minion，比较适合 key 键名统一，value 不同的场景；如果配置到 minion 客户端本地，那就不好统一管理了；
* pillar+nodegroup 方式都是由 salt-master 统一管理，只要主机名符合 nodegroup 里自定义的规范，就可以实现一次配置，多次使用；

如果服务器集群架构更大更复杂，或者需要构建 Web 平台，那肯定就要将配置数据入库了，有机会希望能尝试一下。
