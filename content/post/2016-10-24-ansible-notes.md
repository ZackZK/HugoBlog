---
date: 2016-10-24T00:00:00Z
description: null
modified: 2016-10-24
tags:
- ansible
title: ansible使用问题总结
---

## 1. Network is unreachable问题
问题 ： 使用apt-get在目标主机可正常安装，但是通过ansible的apt模块报 Network is unreachable的问题

原因 ： 目标主机上访问外网需要设置代理， ansible默认不会执行目标主机的.bashrc设置环境变量

解决方法 ： 使用ansible的environment关键字来设置proxy

见官方文档： http://docs.ansible.com/ansible/playbooks_environment.html

官方的例子

{{< highlight bash >}}
- hosts: all
  remote_user: root

  tasks:

    - apt: name=apache2 state=installed
      environment:
      http_proxy: http://proxy.example.com:8080
      
{{< / highlight >}}

也可以使用变量：

{{< highlight bash >}}
- hosts: all
  remote_user: root

  # here we make a variable named "proxy_env" that is a dictionary
  vars:
    proxy_env:
      http_proxy: http://proxy.example.com:8080

  tasks:

    - apt: name=apache2 state=installed
      environment: "{{proxy_env}}"
{{< / highlight >}}

## 2. 安装mongodb

{{< highlight yaml >}}
---
- name: download key
  apt_key: keyserver=hkp://keyserver.ubuntu.com:80 id=EA312927 state=present

- name : 2. update Repo
  apt_repository: repo='deb http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.2 multiverse' state=present

- name: 3. install mongodb
  apt: name=mongodb-org state=present 

- name: 4. start mongodb
  service: name=mongod state=started enabled=yes

- name: 5. create mongodb user
  mongodb_user: database=algo name=algo password=algo state=present
{{< / highlight >}}


运行ansible-playbook到第三步时碰到这样的错误“E: There were unauthenticated packages and -y was used without --allow-unauthenticated”
查看apt模块说明，需要加 allow_unauthenticated=yes 参数

{{< highlight yaml >}}
apt: name=mongodb-org state=present allow_unauthenticated=yes
{{< / highlight >}}

## 3. 编译安装mesos

根据http://mesos.apache.org/gettingstarted/ 的步骤：

1. 下载解压mesos源代码

{{< highlight yaml >}}
- name: download mesos source code
  get_url:
    url: http://archive.apache.org/dist/mesos/1.0.1/mesos-1.0.1.tar.gz
    dest: /var/log/mesos-1.0.1.tar.gz
    force: no

- name: untar mesos source
  unarchive: src=/var/log/mesos-1.0.1.tar.gz dest=/var/log/ remote_src=yes copy=no
{{< / highlight >}}

对于当前使用的ansible 2.1.2.0，
get_url模块经过测试验证如果目标地址是目录的话，force选项即使为no也不会产生任何作用，即如果 “dest: /var/log/” 将mesos代码放在/var/log/目录下，但是未指定目标文件名，虽然force=no，get_url也会重新下载

unarchive模块的remote_src和copy选项冲突的问题，虽然ansible官方的指导手册上说明如果使用 remote_src=yes 的话，就会去远程服务器上访问文件，但是经过验证，在2.1.2.0上，仍然是访问ansible主机上文件。加了copy=no才解决。应该在2.2以后版本解决了该问题


2. 在ubuntu上要安装一系列依赖：

{{< highlight yaml >}}
- name: install System Requirements
  apt: name={{item}} state=present
  with_items:
    - tar
    - wget
    - git
    - openjdk-7-jdk
    - autoconf
    - libtool
    - build-essential
    - python-dev
    - libcurl4-nss-dev
    - libsasl2-dev
    - libsasl2-modules
    - maven
    - libapr1-dev
    - libsvn-dev
    - zlib1g-dev                # 官方文档中没有
{{< / highlight >}}

3. 编译安装mesos

{{< highlight yaml >}}
- name: configure mesos
  command: './configure chdir=/var/log/mesos-1.0.1'

- name: make mesos
  make: chdir=/var/log/mesos-1.0.1
  {{< / highlight >}}

编译过程中碰到maven编译出错，报下载不了pom文件，这个是因为maven不使用系统默认的环境变量。参考https://maven.apache.org/guides/mini/guide-proxies.html, 有两种方式为maven添加代理。这里因为不是直接命令行maven编译，所以采用maven配置文件的方式。创建settings.xml文件，内如如下：
{{< highlight xml >}}
<settings>
  .
  .
  <proxies>
   <proxy>
      <id>example-proxy</id>
      <active>true</active>
      <protocol>http</protocol>
      <host>proxy.example.com</host>
      <port>8080</port>
      <username>proxyuser</username>
      <password>somepassword</password>
      <nonProxyHosts>www.google.com|*.example.com</nonProxyHosts>
    </proxy>
  </proxies>
  .
  .
</settings>
{{< / highlight >}}

将示例中相关参数根据实际情况修改。在编译mesos之前，将该配置文件拷贝到远程主机的 ~/.m2/setting.xml

{{< highlight yaml >}}
- name: add proxy for maven
  template: src=settings.xml dest=settings.xml
{{< / highlight >}}
