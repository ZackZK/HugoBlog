---
date: 2016-11-26T00:00:00Z
description: clojure学习笔记
modified: 2016-12-20
tags:
- clojure
- lisp
title: clojure 笔记
---

# clojure修改maven repo的方法 #
使用国内clojure maven repo
Maven官方库经常非常慢，阿里云也提供maven库镜像，可以在project.clj中添加下面配置使用阿里云repo：

{{< highlight clojure >}}
  :mirrors {"central" {:name "central"
                       :url "http://maven.aliyun.com/nexus/content/groups/public/"}}
{{< / highlight >}}


