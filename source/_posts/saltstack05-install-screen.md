---
title: "SaltStack实践05-screen安装和自定义配置"
date: 2017-11.02 15:52:00
tags: [SaltStack]
---

## 环境说明
OS: CentOS-6.9-x86_64-minimal
SaltStack: Master/Minions version-2017.7.2

## Config in salt-master
```
# /data/salt/pillar/top.sls 
base:
    "*":
      - screen

# /data/salt/pillar/screen/init.sls 
screen_config: 'caption always "%{= kw}%-w%{= kG}%{+b}[%n %t]%{-b}%{= kw}%+w %=%d %M %0c %{g}%H%{-}"'

# /data/salt/base/init/screen/init.sls 
{% set screen_config = pillar.get('screen_config') %}
screen:
  pkg.installed:
    - name: screen

/etc/screenrc:
  file.append:
    - name: /etc/screenrc
    - text: {{ screen_config }}
    - require:
      - pkg: screen
```
<!--more-->
## Test
```
[root@xxx-salt-master ~]# salt 'xxx-salt-minion-01' pillar.get screen_config
xxx-salt-minion-01:
    caption always "%{= kw}%-w%{= kG}%{+b}[%n %t]%{-b}%{= kw}%+w %=%d %M %0c %{g}%H%{-}"
[root@xxx-salt-master ~]# salt 'xxx-salt-minion-01' cmd.run 'rpm -qa|grep screen'
xxx-salt-minion-01:
ERROR: Minions returned with non-zero exit code


[root@xxx-salt-master ~]# salt 'xxx-salt-minion-01' state.sls init.screen
xxx-salt-minion-01:
----------
          ID: screen
    Function: pkg.installed
      Result: True
     Comment: The following packages were installed/updated: screen
     Started: 10:54:57.646091
    Duration: 9544.787 ms
     Changes:   
              ----------
              screen:
                  ----------
                  new:
                      4.0.3-19.el6
                  old:
----------
          ID: /etc/screenrc
    Function: file.append
      Result: True
     Comment: Appended 1 lines
     Started: 10:55:07.197067
    Duration: 6.985 ms
     Changes:   
              ----------
              diff:
                  --- 
                  
                  +++ 
                  
                  @@ -214,3 +214,4 @@
                  
                   # defnonblock 1
                   # blankerprg rain -d 100
                   # idle 30 blanker
                  +caption always "%{= kw}%-w%{= kG}%{+b}[%n %t]%{-b}%{= kw}%+w %=%d %M %0c %{g}%H%{-}"

Summary for xxx-salt-minion-01
------------
Succeeded: 2 (changed=2)
Failed:    0
------------
Total states run:     2
Total run time:   9.552 s

[root@xxx-salt-master ~]# salt 'xxx-salt-minion-01' cmd.run 'rpm -qa|grep screen'
xxx-salt-minion-01:
    screen-4.0.3-19.el6.x86_64
[root@xxx-salt-master ~]# salt 'xxx-salt-minion-01' cmd.run 'tail -n 1 /etc/screenrc'
xxx-salt-minion-01:
    caption always "%{= kw}%-w%{= kG}%{+b}[%n %t]%{-b}%{= kw}%+w %=%d %M %0c %{g}%H%{-}"
```

## 参考资料

