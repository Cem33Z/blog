---
title: "Nginx access_log 不记录特定IP的访问记录"
date: 2017-12-07 11:34:21
tags: [Nginx]
---

## 环境说明
* CentOS Linux release 7.3.1611 (Core)
* nginx version: nginx/1.12.2


## Configure
### 方法一：利用 map 指令实现.
* map 指令是由 ngx_http_map_module 模块提供的，默认自动加载
* access_log 默认使用 combined 的配置来记录访问日志

1. 修改 vhost 配置
```
# /data/app/nginx/conf/conf.d/xxx.conf
map $remote_addr $log_ip {
    "192.168.1.100" 0;
    "10.0.0.100" 0;
    default 1;
}

server {
    ...
    access_log /data/logs/nginx/xxx.access.log combined if=$log_ip;
    ...
}
```

2. 重新加载生效
```
# nginx -t
nginx: the configuration file /data/app/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /data/app/nginx/conf/nginx.conf test is successful
```

### 方法二：利用 if 匹配实现
1. 修改 vhost 配置
```
# /data/app/nginx/conf/conf.d/xxx.conf
server {
    ...
    location  / {
      if ($remote_addr = "192.168.1.100" ) {
        access_log off;
      }
    }
	...
}
```

2. 重新加载生效
```
# nginx -t
nginx: the configuration file /data/app/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /data/app/nginx/conf/nginx.conf test is successful
```

## Test
1. 在客户端浏览器里访问
2. 服务器端打开 xxx.access.log 日志，实时查看来自于客户端 IP 的请求是否被记录 ` tail -f /data/logs/nginx/xxx.access.log `
3. 客户端切换 IP 重试访问，服务器端日志是否正常打印
