---
date: 2015-09-09T00:00:00Z
description: gevent 安装失败
modified: 2015-09-09
tags:
- python
- gevent
title: gevent安装
---

gevent的安装过程：

## 1. 安装greenlet依赖 ##

```Shell
  sudo pip install greenlet
```

## 2. 安装gevent ##

```shell
   sudo pip install gevent
```

安装过程中，碰到下面错误：

```shell
gevent/gevent.core.c:9:22: fatal error: pyconfig.h: No such file or directory

 #include "pyconfig.h"

                      ^

compilation terminated.

error: command 'x86_64-linux-gnu-gcc' failed with exit status 1

----------------------------------------
Cleaning up...
Command /usr/bin/python -c "import setuptools, tokenize;__file__='/tmp/pip_build_root/gevent/setup.py';exec(compile(getattr(tokenize, 'open', open)(__file__).read().replace('\r\n', '\n'), __file__, 'exec'))" install --record /tmp/pip-11RdgV-record/install-record.txt --single-version-externally-managed --compile failed with error code 1 in /tmp/pip_build_root/gevent

Storing debug log for failure in /root/.pip/pip.log

```

这是因为缺少python-dev,安装上即可。

```shell
apt-get install python-dev
```