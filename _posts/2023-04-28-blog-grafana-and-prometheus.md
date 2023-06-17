---
title: '硕士毕业论文系统速成：Grafana和Prometheus'
date: 2023-04-28
permalink: /posts/2023/04/grafana-and-prometheus/
tags:
  - 毕业设计
  - grafana
  - prometheus
---

本文主要介绍我利用grafana和prometheus两个开源系统快速搭建毕业论文前端的实践经验。这两个系统非常易于上手，只需要花费很少的时间进行学习和搭建。以下是我的操作记录，分享出来供大家参考。

我直接在CentOS 7.8操作系统上安装了相关软件，没有使用docker，这样做的好处是方便快捷，但缺点是可能会修改系统环境。

具体的操作步骤如下：

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
- 解压文件并运行Pushgateway：
  ```sh
  tar -zxvf pushgateway-1.4.0.linux-amd64.tar.gz
  cd pushgateway-1.5.1.linux-amd64/
  ./pushgateway
  ```
- 验证Pushgateway服务是否成功启动，访问http://YourIP:9091 ，如果能够访问，说明启动成功；
- 配置Prometheus中的Pushgateway，在Prometheus的配置文件prometheus.yml中添加如下配置项：
  
  ```yaml
  scrape_configs:
    - job_name: 'pushgateway'
      honor_labels: true
      scrape_interval: 5s
      static_configs:
        - targets: ['YourNodeIP:9091']
  ```
- 重新启动Prometheus服务使配置生效。

### 5. 使用Prometheus client库（python）上传数据
- 安装Prometheus client库：
  ```
  pip install prometheus_client
  ```
- 编写Python代码，将JSON数据上传到Pushgateway中。以下是一个示例代码：
  ```python
  from prometheus_client import CollectorRegistry, Gauge, push_to_gateway
  # 假设要上传的数据如下
  data = {
      "sip": "10.0.0.1",
      "dip": "10.0.0.2",
      "latency": 20,
  }
  registry=CollectorRegistry()
  # 创建Gauge对象
  g = Gauge('my_metric', 'A gauge', ['sip', 'dip'], registry=registry)

  # 设置指标值
  g.labels(sip=data["sip"], dip=data["dip"]).set(data["latency"])

  # 推送数据至Pushgateway
  push_to_gateway('YourIP:9091', job='my_job', registry=registry)
  ```

### 6. 使用[PromQL](https://www.prometheus.wang/quickstart/promql_quickstart.html)查询监控数据，在prometheus中进行数据聚合：
- 进入prometheus的graph页面 http://YourIP:9090/graph ：
![探索metrics](https://Yiu-chung.github.io/images/prom_search.png)
  - ① 点击Metrics Explorer，查看有哪些metrics；
  - ② 选择一个metric， 如node_memory_Active_bytes；
  - ③ 点击Execute，查看metric的时序曲线。

- 按照时间（每5分钟）聚合（按照标签区分时间序列）：
  ```sh
  avg_over_time(YourMetric[5m])
  ```

- 不区分标签聚合：
  ```sh
  avg(YourMetric)
  ```

### 7. 将数据接入到Grafana中，设置Dashboard
- 设置Data Source/Prometheus： Configuration $\rightarrow$ Data Source $\rightarrow$ Prometheus；
- 填入prometheus的URL；
- 保存：Save & test。
![添加data source](https://Yiu-chung.github.io/images/grafana_datasource.png)
- New Dashboard $\rightarrow$ Add a new panel
- 编辑一个panel：
![编辑panel](https://Yiu-chung.github.io/images/edit_panel.png)
  - ① 使用PromQL查询数据；
  - ② 运行查询，获得指标时序曲线；
  - ③ 编辑title；
  - ④ 应用。
- 保存Dashboard。
  
### 8. 在Grafana中设置告警
- 进入Dashboard，Edit Panel；
- 点击Alert $\rightarrow$ Create alert rule from this panel；
- 选择Classic_conditions，设置针对最新数据last()的告警阈值：
![设置alert](https://Yiu-chung.github.io/images/grafana_alert.png)
- 填写告警所属的Folder 和group，保存。 