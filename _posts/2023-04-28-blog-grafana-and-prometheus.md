---
title: '硕士毕业论文系统速成：Grafana和Prometheus'
date: 2023-04-28
permalink: /posts/2023/04/grafana-and-prometheus/
tags:
  - cool posts
  - category1
  - category2
---

本文主要介绍我使用grafana和prometheus两个开源系统速成毕业论文相关系统前端的实践。对于这两个系统的学习和搭建我只花费了一天左右的时间，极易上手。下面将搭建步骤和简单的时间操作记录如下，方便学习交流。

我直接在操作系统为centos7.8的主机上安装相关软件，没有使用docker，更加方便直接，缺点是可能会修改系统环境。

整体包括以下几步：
### 1. 安装grafana
- 在CentOS上安装Grafana的yum源：
  ```
  sudo tee /etc/yum.repos.d/grafana.repo <<EOF
  [grafana]
  name=grafana
  baseurl=https://packages.grafana.com/oss/rpm
  repo_gpgcheck=1
  enabled=1
  gpgcheck=1
  gpgkey=https://packages.grafana.com/gpg.key
  sslverify=1
  sslcacert=/etc/pki/tls/certs/ca-bundle.crt
  EOF

  ```
- 安装 Grafana：
  ```
  sudo yum install grafana

  ```
- 启动Grafana服务：
  ```
  sudo systemctl start grafana-server

  ```
- 在浏览器中通过3000端口访问该服务，首次登陆默认用户和密码都是admin，登录后可修改密码提高安全性：
  ```
  http://yourIP:3000
  ```
  ![](https://Yiu-chung.github.io/images/grafana_login.png)
- 如果无法访问，有可能是防火墙问题，需要打开相应的端口：
  ```
  sudo firewall-cmd --add-port=3000/tcp --permanent
  sudo firewall-cmd --reload
  ```

Headings are cool
======

You can have many headings
======

Aren't headings cool?
------