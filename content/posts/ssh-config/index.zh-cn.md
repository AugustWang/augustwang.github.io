---
weight: 1
title: "使用SSH配置有效管理多台服务器"
date: 2025-11-15T17:35:33+08:00
lastmod: 2025-11-15T17:35:33+08:00
draft: false
author: "August"
authorLink: "https://augustwang.github.io"
description: "使用SSH配置有效管理多台服务器."
images: []
resources:
- name: "featured-image"
  src: "featured-image.jpeg"

tags: ["content"]
categories: ["documentation"]

lightgallery: true
---

这篇文章主要介绍如何使用 SSH 配置来有效管理多台服务器的.

<!--more-->

## 1 场景

作为 Linux 系统的使用者, 我们需要管理多个服务器，这些服务器有不同的应用场景 -- 线上环境、测试环境、开发环境等等。

每次 SSH 登录到服务器时, 都需要输入一串命令如：
```bash
ssh user@host -p port -i /path/to/key
```
而且每台服务器的登录信息都可能不同，比如用户名、密码、端口、密钥文件等等。

这样在实际的使用中，就会很麻烦，很影响使用效率。

那么有没有更好的方式来管理多个服务器呢？答案是：SSH 配置文件 `~/.ssh/config`。

当然，也可以选择使用其他方案或工具来管理，比如：xshell, 自定义脚本等等。

但是，SSH 配置文件 `~/.ssh/config` 是最常用的一种方式，下面就主要介绍如何使用 SSH 配置文件 `~/.ssh/config` 来管理多个服务器。

## 2 SSH 配置文件 `~/.ssh/config`

SSH 配置文件 `~/.ssh/config` 是一个用于管理 SSH 连接的配置文件，它可以用来保存多个服务器的登录信息，并使用一个命令来登录多个服务器。

### 2.1 SSH 配置文件参数定义

创建 SSH 配置文件 `~/.ssh/config`，配置参数示例如下：
```bash
Host server1
  HostName 192.168.1.1
  User root
  Port 22
  IdentityFile ~/.ssh/id_rsa
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  LogLevel ERROR
  PubkeyAuthentication yes
  PasswordAuthentication no
  PreferredAuthentications publickey
  ForwardAgent yes
  Ciphers aes128-ctr,aes192-ctr,aes256-ctr,aes128-gcm@openssh.com,aes256-gcm@openssh.com,chacha20-poly1305@openssh.com
  Compression yes
  ServerAliveInterval 60
  ServerAliveCountMax 3
  TCPKeepAlive yes
  TCPKeepAliveInterval 60
  ConnectTimeout 10
  ControlMaster auto
  ControlPath ~/.ssh/%C
  ControlPersist 10m
  ProxyCommand nc -X 5 -x 127.0.0.1:1080 %h %p
  LocalForward 8080 localhost:80
  RemoteForward 80 localhost:8080
  ProxyJump jump1,jump2
```

参数描述如下：

- `Host` 段用于定义一个服务别名，
- `HostName` 用于定义服务对应的主机名，
- `User` 用于定义服务对应的用户名，
- `Port` 用于定义服务对应的端口，
- `IdentityFile` 用于定义服务对应的密钥文件，
- `StrictHostKeyChecking` 用于定义是否检查服务对应的主机密钥，
- `UserKnownHostsFile` 用于定义服务对应的主机密钥文件，
- `LogLevel` 用于定义服务对应的日志级别，
- `PubkeyAuthentication` 用于定义是否使用密钥进行认证，
- `PasswordAuthentication` 用于定义是否使用密码进行认证，
- `PreferredAuthentications` 用于定义优先的认证方式，
- `ForwardAgent` 用于定义是否转发 SSH 代理，
- `Ciphers` 用于定义 SSH 连接使用的密码学，
- `Compression` 用于定义是否压缩 SSH 连接，
- `ServerAliveInterval` 用于定义 SSH 连接空闲时间，
- `ServerAliveCountMax` 用于定义 SSH 连接空闲时间后，允许的最大空闲次数，
- `TCPKeepAlive` 用于定义是否启用 TCP KeepAlive，
- `TCPKeepAliveInterval` 用于定义 TCP KeepAlive 的间隔时间，
- `ConnectTimeout` 用于定义 SSH 连接超时时间，
- `ControlMaster` 用于定义是否启用 SSH 控制，
- `ControlPath` 用于定义 SSH 控制路径，
- `ControlPersist` 用于定义 SSH 控制保持时间，
- `ProxyCommand` 用于定义 SSH 代理命令，
- `LocalForward` 用于定义本地端口转发，
- `RemoteForward` 用于定义远程端口转发。
- `ProxyJump` 用于定义 SSH 代理跳转。

### 2.2 SSH 配置管理多台服务器

现在假设我有一台服务器，public ip为192.168.1.1，用户名为root，密钥文件为`/root/.ssh/id_rsa`

使用ssh命令登录服务器：
```bash
ssh root@192.168.1.1 -i /root/.ssh/id_rsa
```

以上命令如果在 `~/.ssh/config` 中配置则是：
```config
Host server1
  HostName 192.168.1.1
  User root
  Port 22
  IdentityFile /root/.ssh/id_rsa
```

配置好 `~/.ssh/config` 后，就可以使用简写的服务别名来登录，命令为：
```bash
ssh server1
```

这样如果是多台服务器，只需要在 `~/.ssh/config` 中添加对应服务别名即可

### 2.3 SSH 配置堡垒机访问内网服务器

假设有一个堡垒机，堡垒机ip为192.168.1.2，用户名为root，密钥文件为`/root/.ssh/id_rsa` 

添加堡垒机配置：
```config
Host server1
  HostName 192.168.1.1
  User root
  Port 22
  IdentityFile /root/.ssh/id_rsa
  ProxyJump server2

Host server2
  HostName 192.168.1.2
  User root
  Port 22
  IdentityFile /root/.ssh/id_rsa
```

现在这份配置中，server2 是 server1 的堡垒机，server1 可以通过 server2 登录。

注意的是，堡垒机配置的HostName是外网的，而server1的HostName可以是和server2同网络的内网ip，这样能够直接登录内网的服务器，有效的提高安全性。

### 2.4 SSH 配置全局设置

在管理多台服务器时，肯定有很多参数是一致的，比如：用户名、密钥文件、端口等等。

对于重复的参数，我们可以设置全局的配置，全局配置的优先级最高，比如：
```config
Host *
  User root
  IdentityFile /root/.ssh/id_rsa
  Port 22

Host server1
  HostName 192.168.1.1
  ProxyJump server2

Host server2
  HostName 192.168.1.2
```
这样就不需要在每个服务器的配置中重复设置用户名、密钥文件、端口了。

### 2.5 SSH 配置分组管理

在管理多台服务器时，每个环境都可能存在多个服务器组，这样就可以把这些登录配置相同的，

使用通配符`*`来管理匹配多个服务器组，比如：`dev-*`、`test-*`等等。

可以使用分组管理，比如：
```config
Host dev-*
  HostName 192.168.1.1
  User root
  IdentityFile /root/.ssh/id_rsa
  Port 22

Host test-*
  HostName 192.168.1.2
  User root
  IdentityFile /root/.ssh/id_rsa
  Port 22
```

更多配置：[SSH 配置高手](https://mp.weixin.qq.com/s/mQVpCcjipUZ01r7UuYCThA)