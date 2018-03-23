---
title: "SaltStack实践07-SSH服务配置"
date: 2017-11.02 16:22:00
tags: [SaltStack]
---
## 环境说明
OS: CentOS-6.9-x86_64-minimal
SaltStack: Master/Minions version-2017.7.2

<!--more-->
## Config in salt-master
```
# /data/salt/pillar/sshd/init.sls 
firewall:
  sshd_firewall:
    port: 22
    allow:
      - 192.168.0.0/24

# /data/salt/base/init/sshd/init.sls 
sshd_config:
 file.managed:
    - name: /etc/ssh/sshd_config
    - user: root
    - group: root
    - mode: 600
    - source: salt://init/sshd/files/etc/ssh/sshd_config
    - template: jinja

# /data/salt/base/init/sshd/files/etc/ssh/sshd_config 
{% set sshd_port = pillar['firewall']['sshd_firewall']['port'] %}
{% set sshd_host = grains['ipv4'][1] %}

Port {{ sshd_port }}

ListenAddress {{ sshd_host }}
Protocol 2

HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_dsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key

UsePrivilegeSeparation yes


KeyRegenerationInterval 3600
ServerKeyBits 768

SyslogFacility AUTH
LogLevel VERBOSE

LoginGraceTime 120

StrictModes yes

RSAAuthentication yes
PubkeyAuthentication yes

IgnoreRhosts yes

RhostsRSAAuthentication no

HostbasedAuthentication no

PermitEmptyPasswords no

ChallengeResponseAuthentication no

PasswordAuthentication yes

X11Forwarding no
X11DisplayOffset 10
X11UseLocalhost yes
PrintMotd no
PrintLastLog yes
TCPKeepAlive yes

AcceptEnv LANG LC_*

Subsystem sftp /usr/lib/openssh/sftp-server

UsePAM yes
```

### Test
```
[root@xxx-salt-master ~]# salt 'xxx-salt-minion-01' state.sls init.sshd

[root@xxx-salt-master ~]# salt 'xxx-salt-minion-01' cmd.run "grep -E '^Port|^Listen' /etc/ssh/sshd_config"
xxx-salt-minion-01:
    Port 22
    ListenAddress 192.168.0.52
```

## 参考资料

