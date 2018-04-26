---
date: 2015-09-09T00:00:00Z
description: 介绍spark的相关配置和使用
modified: 2015-09-09
tags:
- spark
- spark configuration
title: spark 配置及使用
---

## spark配置 ##
spark的配置文件放在conf目录下，默认会有类似xx.template的模板文件，去掉后缀.template就可以作为spark相应的配置文件。

### spark 在阿里云上的配置 ##
在一般的云服务器上一般会有两个ip地址，比如阿里云主机上，eth0默认是内部网络ip（用于阿里云主机之间通信），eth1才是外网可访问的ip地址。
spark会默认会使用8080端口作为UI端口号，并且绑定在0.0.0.0上，也就意味着外部网络可以通过 http://eth1_ip:8080访问spark UI界面，
如果要改变端口号，在conf/spark-env.sh文件中加入下面内容即可

```shell
export SPARK_MASTER_WEBUI_PORT=8090
```

如果当前指定的端口号被别的应用程序占用，spark自动会继续加1往下找到一个可用的端口号。

### Spark的log级别设置 ###
默认情况下，spark会在屏幕打印INFO级别以上的log，这些信息对于调试来说很重要，但是当spark程序正常运行时，可能不需要这么多信息，这个时候，可以通过修改
conf/log4j.properties：
在文件中找到该行：

```shell
log4j.rootCategory=INFO, console
```

将其修改成

```shell
log4j.rootCategory=WARN, console
```


