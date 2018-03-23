---
title: "SaltStack实践04-用户管理&sudo权限"
date: 2017-11.02 14:02:00
tags: [SaltStack]
---

## 环境说明
OS: CentOS-6.9-x86_64-minimal
SaltStack: Master/Minions version-2017.7.2

## 处理思路
1. 将用户信息配置在 pillar 里；
2. state 通过 pillar.get 获取用户信息，执行相关操作；
3. 创建组、创建用户、修改密码、增加 sudo 权限

<!--more-->
## Congfig reactor
### 关键配置文件的内容
#### Pillar config
```
# /data/salt/pillar/top.sls 
base:
    "*":
      - users

# /data/salt/pillar/users/init.sls
manager_user_list:
  xdx:
    fullname: 'Test1'
    uid: 1000
    gid: 1000
    group: manager
    passwd: '$1$xdx$5x/o2OSo0vLpLlbtIAHd/0'
    shell: /bin/bash
    sudo: True
  dxd:
    fullname: 'Test2'
    uid: 1001
    gid: 1000
    group: manager
    passwd: '$1$dxd$5x/o2OSo0vLpLlbtIAHd/0'
    shell: /bin/bash
    sudo: True
```

#### State cofnig
```
# /data/salt/base/users/init.sls 
{% set manager_users = pillar.get('manager_user_list', {}).items() %}
{% for user, args in manager_users %}
{{ user }}:
  group.present:
    - name: {{ args['group'] }}
    - gid: {{ args['gid'] }}
  user.present:
    - home: /home/{{ user }}
    - shell: {{ args['shell'] }}
    - uid: {{ args['uid'] }}
    - gid: {{ args['gid'] }}
    - fullname: {{ args['fullname'] }}
    {% if 'passwd' in args %}
    - password: {{ args['passwd'] }}
    {% endif %}
    - require:
      - group: {{ user }}
{% if 'sudo' in args and args['sudo'] %}
sudoer-{{ user }}:
  file.managed:
    - name: /etc/sudoers.d/{{ user }}
    - contents:
      - '{{ user }}  ALL=(ALL)    ALL'
    - require:
      - user: {{ user }}
{% else %}
  file.absent:
    - name: /etc/sudoers.d/{{ user }}
{% endif %}
{% endfor %}
```

## Test
```
[root@xxx-salt-master users]# salt 'xxx-salt-minion-01' state.sls users
```

## sudo权限

