---
title: "SaltStack实践11-使用Salt-ssh安装minions"
date: 2017-11.03 12:26:00
tags: [SaltStack]
---

## 环境说明
OS: CentOS-6.9-x86_64-minimal
SaltStack: Master/Minions version-2017.7.2

## 处理思路
1. salt-master 先安装 salt-ssh 工具，测试确保能通过 salt-ssh 连接到需要安装 salt-minion 的新机器；
2. 利用 salt-ssh 给新机器安装 Saltstack 的 pkgrepo；
3. 利用 salt-ssh 给新机器安装指定版本的 salt-minion；
4. 通过 pillar.get 获取 salt-master 的 IP 后写入到 `salt://minions/files/etc/salt/minion` 文件；
5. 推送到 `salt://minions/files/etc/salt/minion` 文件给新机器的 `/etc/salt/minion`；
6. 连接到 salt-master，操作完成；

<!--more-->

## Install salt-ssh
```
[root@xxx-salt-master ~]# yum install -y salt-ssh
[root@xxx-salt-master ~]# rpm -qa | grep salt-ssh
salt-ssh-2017.7.2-1.el6.noarch
```

### 配置文件
```
# /etc/salt/master.d/rosters_include.conf 
roster_file: /data/salt/roster.d/rosters

# /data/salt/roster.d/rosters 
xxx-salt-minion-02:
  host: 192.168.0.53
  user: root
  passwd: 123456
  port: 22
  timeout: 10
```

### Test-connection
```
[root@xxx-salt-master ~]# salt-ssh 'xxx-salt-minion-02' test.ping
xxx-salt-minion-02:
    ----------
    retcode:
        254
    stderr:
    stdout:
        The host key needs to be accepted, to auto accept run salt-ssh with the -i flag:
        The authenticity of host '192.168.0.53 (192.168.0.53)' can't be established.
        RSA key fingerprint is 37:b0:6b:9a:98:60:12:f8:e3:7c:d0:1b:e8:ef:20:63.
        Are you sure you want to continue connecting (yes/no)?
```

提示 (yes/no) 确认，这是因为新机器 SSH 初始连接的时候，会确认添加 RSA key fingerprint 这一步，可以用 `-i`参数忽略

```
[root@xxx-salt-master ~]# salt-ssh -i 'xxx-salt-minion-02' test.ping
xxx-salt-minion-02:
    True

```

## Config
### 在 pillar 里配置一个 salt-master_ip 的项
```
# /data/salt/pillar/top.sls 
base:
    "*":
      - base

# /data/salt/pillar/base.sls 
salt_master_host: 192.168.0.55

# 测试可以在 xxx-salt-minion-02 获取到 salt_master_host 的值
[root@xxx-salt-master minions]# salt-ssh -i 'xxx-salt-minion-02' pillar.get salt_master_host
xxx-salt-minion-02:
    192.168.0.55
```

### 关键配置文件的内容
```
# /data/salt/base/minions/
/data/salt/base/minions/
├── files
│   └── etc
│       └── salt
│           └── minion
├── install.sls
└── repo.sls

# /data/salt/base/minions/install.sls 
#SaltStack repo
include:
  - minions.repo

#salt_minion_install
minion_install:
  pkg.installed:
    - pkgs:
      - salt-minion
    - unless: rpm -qa | grep salt-minion
minion_conf:
  file.managed:
    - name: /etc/salt/minion
    - source: salt://minions/files/etc/salt/minion
    - user: root
    - group: root
    - mode: 640
    - template: jinja
    - defaults:
      minion_id: {{ grains['fqdn_ip4'][0]}}
    - require:
      - pkg: minion_install
minion_service:
  service.running:
    - name: salt-minion
    - enable: True
    - require:
      - file: minion_conf 

# /data/salt/base/minions/repo.sls 
pkgrepo:
  pkgrepo.managed:
    - humanname: 'SaltStack repo for RHEL/CentOS $releasever'
    - name: 'saltstack-repo'
    - baseurl: https://repo.saltstack.com/yum/redhat/$releasever/$basearch/archive/2017.7.2
    - enabled: 1
    - gpgcheck: 1
    - gpgkey: https://repo.saltstack.com/yum/redhat/$releasever/$basearch/archive/2017.7.2/SALTSTACK-GPG-KEY.pub

  cmd.run:
    - name: yum clean expire-cache 

# /data/salt/base/minions/files/etc/salt/minion 
master: {{pillar.get('salt_master_host')}}
```

## Install salt-minion
```
[root@xxx-salt-master ~]# salt-ssh -i 'xxx-salt-minion-02' state.sls minions.install
xxx-salt-minion-02:
----------
          ID: pkgrepo
    Function: pkgrepo.managed
        Name: saltstack-repo
      Result: True
     Comment: Configured package repo 'saltstack-repo'
     Started: 10:43:17.849084
    Duration: 1103.526 ms
     Changes:   
              ----------
              repo:
                  saltstack-repo
----------
          ID: pkgrepo
    Function: cmd.run
        Name: yum clean expire-cache
      Result: True
     Comment: Command "yum clean expire-cache" run
     Started: 10:43:18.955403
    Duration: 172.129 ms
     Changes:   
              ----------
              pid:
                  4423
              retcode:
                  0
              stderr:
              stdout:
                  Loaded plugins: fastestmirror
                  Cleaning repos: base extras saltstack-repo updates
                  6 metadata files removed
----------
          ID: minion_install
    Function: pkg.installed
      Result: True
     Comment: The following packages were installed/updated: salt-minion
     Started: 10:43:19.145060
    Duration: 79134.044 ms
     Changes:   
              ----------
......（省略）
----------
          ID: minion_conf
    Function: file.managed
        Name: /etc/salt/minion
      Result: True
     Comment: File /etc/salt/minion updated
     Started: 10:44:38.281732
    Duration: 90.083 ms
     Changes:   
              ----------
              diff:
                  ---  
                  +++
......（省略）
                  +master: 192.168.0.55
----------
          ID: minion_service
    Function: service.running
        Name: salt-minion
      Result: True
     Comment: Service salt-minion has been enabled, and is running
     Started: 10:44:38.376622
    Duration: 7886.365 ms
     Changes:   
              ----------
              salt-minion:
                  True

Summary for xxx-salt-minion-02
------------
Succeeded: 5 (changed=5)
Failed:    0
------------
Total states run:     5
Total run time:  88.386 s
```

### Accept keys
```
# xxx-salt-minion-02 的 Keys 请求已经过来了
[root@xxx-salt-master ~]# salt-key -L
Accepted Keys:
xxx-salt-minion-01
Denied Keys:
Unaccepted Keys:
xxx-salt-minion-02
Rejected Keys:

# 允许通过
[root@xxx-salt-master ~]# salt-key -y -a xxx-salt-minion-02
The following keys are going to be accepted:
Unaccepted Keys:
xxx-salt-minion-02
Key for minion xxx-salt-minion-02 accepted.
```
## Test
```
# test.ping 测试连通性没问题
[root@xxx-salt-master ~]# salt 'xxx-salt-minion-02' test.ping
xxx-salt-minion-02:
    True
```

## 遇到的问题
### ImportError: No module named backports.ssl_match_hostname
```
1. salt-ssh 执行的时候报错：ImportError: No module named backports.ssl_match_hostname
[root@xxx-salt-master ~]# salt-ssh -i 'xxx-salt-minion-02' test.ping
xxx-salt-minion-02:
    ----------
    retcode:
        0
    stderr:
        Traceback (most recent call last):
          File "/var/tmp/.root_62a99a_salt/salt-call", line 15, in <module>
            salt_call()
          File "/var/tmp/.root_62a99a_salt/py2/salt/scripts.py", line 386, in salt_call
            import salt.cli.call
          File "/var/tmp/.root_62a99a_salt/py2/salt/cli/call.py", line 6, in <module>
            from salt.utils import parsers
          File "/var/tmp/.root_62a99a_salt/py2/salt/utils/parsers.py", line 28, in <module>
            import salt.config as config
          File "/var/tmp/.root_62a99a_salt/py2/salt/config/__init__.py", line 100, in <module>
            _DFLT_IPC_WBUFFER = _gather_buffer_space() * .5
          File "/var/tmp/.root_62a99a_salt/py2/salt/config/__init__.py", line 90, in _gather_buffer_space
            import salt.grains.core
          File "/var/tmp/.root_62a99a_salt/py2/salt/grains/core.py", line 47, in <module>
            import salt.utils.dns
          File "/var/tmp/.root_62a99a_salt/py2/salt/utils/dns.py", line 28, in <module>
            import salt.modules.cmdmod
          File "/var/tmp/.root_62a99a_salt/py2/salt/modules/cmdmod.py", line 34, in <module>
            import salt.utils.templates
          File "/var/tmp/.root_62a99a_salt/py2/salt/utils/templates.py", line 31, in <module>
            import salt.utils.http
          File "/var/tmp/.root_62a99a_salt/py2/salt/utils/http.py", line 41, in <module>
            import salt.loader
          File "/var/tmp/.root_62a99a_salt/py2/salt/loader.py", line 26, in <module>
            import salt.utils.event
          File "/var/tmp/.root_62a99a_salt/py2/salt/utils/event.py", line 70, in <module>
            import tornado.iostream
          File "/var/tmp/.root_62a99a_salt/py2/tornado/iostream.py", line 39, in <module>
            from tornado.netutil import ssl_wrap_socket, ssl_match_hostname, SSLCertificateError, _client_ssl_defaults, _server_ssl_defaults
          File "/var/tmp/.root_62a99a_salt/py2/tornado/netutil.py", line 51, in <module>
            import backports.ssl_match_hostname
        ImportError: No module named backports.ssl_match_hostname
    stdout:

2. 运行原始系统 SHELL 命令正常
[root@xxx-salt-master ~]# salt-ssh -i 'xxx-salt-minion-02' -r 'uptime'
xxx-salt-minion-02:
    ----------
    retcode:
        0
    stderr:
    stdout:
        root@192.168.0.53's password: 
        xxx-salt-minion-02

解决办法：

1. 给 xxx-salt-minion-02 服务器安装 python-requests 包
[root@xxx-salt-master ~]# salt-ssh -i 'xxx-salt-minion-02' -r 'yum install -y python-requests'

2. 重试：
[root@xxx-salt-master ~]# salt-ssh -i 'xxx-salt-minion-02' test.ping
xxx-salt-minion-02:
    True
```
