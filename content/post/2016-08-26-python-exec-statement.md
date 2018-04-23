---
date: 2016-08-26T00:00:00Z
description: null
modified: 2016-08-26
tags:
- python
- exec
title: python exec的使用过程中碰到的坑
---

python的exec可以直接运行一个文件中的python脚本，或者一个字符串里的python语句。为什么不直接调用这个文件，或者语句呢？一个典型的场景就是我们要运行的python脚本存在数据库中，当然也可以把文件内容从数据库中读取出来，生成临时文件，再运行。但是如果使用exec，就可以on fly的运行python脚本了。

# exec 语法 #

