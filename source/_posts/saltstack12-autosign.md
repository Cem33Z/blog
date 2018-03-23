---
title: "SaltStack实践12-AutoSign自动注册"
date: 2017-11.03 13:28:00
tags: [SaltStack]
---

## 环境说明
OS: CentOS-6.9-x86_64-minimal
SaltStack: Master/Minions version-2017.7.2

## 处理思路
1. salt-master 利用 salt-ssh 给新机器安装 salt-minion 客户端；
2. 新机器的 salt-minion 启动后连接到 salt-master；
3. salt-master 自动允许新机器的 Keys，并推送 top.sls 初始配置给新机器；
4. 新机器同步配置，操作完成；

<!--more-->
## Congfig reactor
### 关键配置文件的内容
```
# /etc/salt/master.d/reactor.conf 
reactor:
  - 'salt/auth':
    - /data/salt/reactor/auth-pending.sls
  - 'salt/minion/xxx-*/start':
    - /data/salt/reactor/auth-complete.sls

# /data/salt/reactor/auth-complete.sls 
{# When an xxx server connects, run state.apply. #}
highstate_run:
  local.state.apply:
    - tgt: {{ data['id'] }}
    - ret: smtp 

# /data/salt/reactor/auth-pending.sls 
{# Xxx server failed to authenticate -- remove accepted key #}
{% if not data['result'] and data['id'].startswith('xxx-') %}
minion_remove:
  wheel.key.delete:
    - args:
      - match: {{ data['id'] }}
minion_rejoin:
  local.cmd.run:
    - tgt: {{ data['id'] }}
    - args:
      - cmd: ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no "{{ data['id'] }}" 'sleep 10 && /etc/init.d/salt-minion restart'
{% endif %}

{# Xxx server is sending new key -- accept this key #}
{% if 'act' in data and data['act'] == 'pend' and data['id'].startswith('xxx-') %}
minion_add:
  wheel.key.accept:
    - args:
      - match: {{ data['id'] }}
{% endif %}
```

## Install salt-minion
利用 salt-ssh 给新机器安装 minion，可以参考“SaltStack实践12-使用Salt-ssh安装minions”

## Test
1. 新安装 salt-minion 的机器第一次连接 salt-master 时，测试是否自动同步配置；
2. 重启 salt-minion 客户端后，测试是否可以同步配置；
3. 当出现冲突的 Keys 时，salt-master 测试是否可以删除旧 Keys；

### 实时获取 Event 的日志信息
```
salt-run state.event pretty=True
```

