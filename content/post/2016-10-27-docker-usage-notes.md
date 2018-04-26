---
date: 2016-10-27T00:00:00Z
description: docker 笔记
modified: 2016-10-27
tags:
- docker
- proxy
- 代理
title: docker 使用笔记
---

## 1. docker下载image设置代理proxy

``` shell
$ docker pull busybox

Using default tag: latest
Pulling repository docker.io/library/busybox
Network timed out while trying to connect to https://index.docker.io/v1/repositories/library/busybox/images. You may want to check your internet connection or if you are behind a proxy.
```

如果系统是用systemd启动docker的daemon的话，docker不会使用系统默认的代理，需要做如下操作，参考https://docs.docker.com/engine/admin/systemd/#http-proxy：

1\. 为docker创建systemd配置文件夹    

```shell
$ mkdir /etc/systemd/system/docker.service.d
```

2\. 创建 /etc/systemd/system/docker.service.d/http-proxy.conf 包含下面内容:    

``` shell
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:80/"
Environment="HTTPS_PROXY=https://proxy.example.com:80/"

```

对于不想使用代理的域名ip地址，使用NO_PROXY关键字

```shell
Environment="HTTP_PROXY=http://proxy.example.com:80/" "NO_PROXY=localhost,127.0.0.1,docker-registry.somecorporation.com"
```

3\. systemd重新加载

``` shell
$ sudo systemctl daemon-reload
```

4\. 查看配置是否生效

``` shell
$ systemctl show --property=Environment docker
Environment=HTTP_PROXY=http://proxy.example.com:80/
```

5\. 重启docker

```shell
$ sudo systemctl restart docker
```


## 2. docker为container设置代理proxy

docker的container不会使用docker daemon的代理，需要额外配置，两种方法，一种是container的命令行加入： --env http_proxy=...来设置，另外是在Dockerfile中指定环境变量：

```shell
FROM ubuntu:14.04
ENV http_proxy <HTTP_PROXY>
ENV https_proxy <HTTPS_PROXY>
RUN apt-get update && apt-get upgrade
```