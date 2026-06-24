---
title: "本地 STF 内网穿透公网访问"
date: 2020-11-25T11:33:40+08:00
---

> 有些事不能靠别人，更多的是靠自己

## 前提需要有一台公网服务器，ip在以下配置中称为 server_ip

## 1、local方式（适用本地部署stf，公网访问）

### 内网穿透配置，使用frp或其他内网穿透工具

- frp下载地址 https://github.com/fatedier/frp/releases
- 服务端配置文件，启动`./frps -c ./frps.ini`

```
# frps.ini

[common]
bind_port = 7000
authentication_method = token
token = your_token
allow_ports = 7400-7600,7100,7110
vhost_http_port = 7100
```

- 客户端配置文件，启动`./frpc -c ./frpc.ini`

```
# frpc.ini

[common]
server_addr = server_ip
server_port = 7000
token = your_token

[stf_web]
type = http
local_port = 7100
custom_domains = server_ip_or_domain

[range:stf]
type = tcp
local_ip = 127.0.0.1
local_port = 7400-7420 # 取决于设备数量，约2口/台
remote_port = 7400-7420 # 取决于设备数量

[stf_websocket]
type = tcp
local_ip = 127.0.0.1
local_port = 7110
remote_port = 7110
```

### stf启动

- 本地启动

```
stf local --public-ip server_ip_or_domain --allow-remote
```
- 访问 server_ip_or_domain:7100
