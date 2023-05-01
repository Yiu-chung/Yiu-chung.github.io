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
  ```sh
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
  ```sh
  sudo yum install grafana
  ```
- 启动Grafana服务：
  ```sh
  sudo systemctl start grafana-server
  ```
- 在web浏览器中通过http://YourIP:3000 访问该服务，首次登陆默认用户和密码都是admin，登录后可修改密码提高安全性：

  ![grafana登录页](https://Yiu-chung.github.io/images/grafana_login.png)
- 如果无法访问，有可能是防火墙问题，需要打开相应的端口：
  ```
  sudo firewall-cmd --add-port=3000/tcp --permanent
  sudo firewall-cmd --reload
  ```

### 2. 安装并运行prometheus
- 下载二进制文件：
  ```sh
  wget https://github.com/prometheus/prometheus/releases/download/v2.33.1/prometheus-2.33.1.linux-amd64.tar.gz
  tar xvfz prometheus-2.33.1.linux-amd64.tar.gz
  cd prometheus-2.33.1.linux-amd64
  ```
- 编辑prometheus.yml文件配置prometheus，暂时使用默认配置即可；
- 启动prometheus：
  ```sh
  ./prometheus --config.file=prometheus.yml
  ```
- 在web浏览器中通过http://YourIP:9090 访问prometheus：
  ![prometheus主页](https://Yiu-chung.github.io/images/prometheus_home.png)
- 同上，如有防火墙，需开放相应端口：
  ```sh
  sudo firewall-cmd --add-port=9090/tcp --permanent
  sudo firewall-cmd --reload
  ```

### 3. 使用node_exporter监控节点 (optinal)
使用node_exporter可以监控机器的负载情况，可以监控节点的CPU、memory等的使用情况。 
- 下载node_exporter的二进制文件：
  ```sh
  wget https://github.com/prometheus/node_exporter/releases/download/v1.2.2/node_exporter-1.2.2.linux-amd64.tar.gz
  tar zxvf node_exporter-1.2.2.linux-amd64.tar.gz
  cd node_exporter-1.2.2.linux-amd64.tar.gz
  ```
- 启动node_exporter：
  ```sh
  ./node_exporter
  ```
- 配置prometheus.yml文件，将node_exporter监控的数据送往prometheus。在prometheus.yml中scrape_configs下<font color="#dddd00">增加</font>以下内容：
  
  ```yaml
  scrape_configs:
    - job_name: 'node'
      static_configs:
        - targets: ['YourNodeIP:9100']
  ```
- 访问prometheus的主页http://YourIP:9090 ，

### 4. 在服务器上安装Pushgateway用于接收数据
- 下载Pushgateway二进制文件（选择合适的版本）：
  ```sh
  wget https://github.com/prometheus/pushgateway/releases/download/v1.5.1/pushgateway-1.5.1.linux-amd64.tar.gz
  ```

Headings are cool
======

You can have many headings
======

Aren't headings cool?
------