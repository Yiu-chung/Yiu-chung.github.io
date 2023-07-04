---
title: '网络性能监控原型系统搭建'
date: 2023-06-12
permalink: /posts/2023/06/net-monitor-system/
tags:
  - 网络性能
  - 端到端测量
  - 监控系统
---

本文记录搭建一个端到端网络性能监控的原型系统的步骤。搭建环境为几台centos。

具体的操作步骤如下：
### 0. 快速部署
#### a. 主控节点
- 在控制节点上解压NetMonitor.zip文件：
  ```sh
  unzip NetMonitor.zip
  ```
- 进入目录：
  ```sh
  cd NetMonitor
  ```
- 启动prometheus和pushgateway：
  ```sh
  ./prometheus/prometheus --config.file=./prometheus/prometheus.yml
  ./pushgateway/pushgateway
  ```
- 启动任务配置服务：
  ```sh
  node server.js
  ```
- 启动任务监控：
  ```sh
  python3 filehandler.py
  ```

#### b. 测量节点
- 在测量节点上解压NetMonitorNode.zip文件：
  ```sh
  unzip NetMonitorNode.zip
  ```
- 进入目录：
  ```sh
  cd NetMonitorNode
  ```
- 启动PerfTrace：
  ```sh
  ./PerfTrace/perftrace_srv
  ```
- 启动测量代理
  ```sh
  python3 mAgent.py
  ```

## 说明：具体见步骤1-7

### 1. 安装Node、Express
- 安装nodejs：
  ```sh
  sudo yum install -y epel-release
  sudo yum install -y nodejs
  ```
- 查看node版本
  ```sh
  node -v
  ```
- 安装npm
  ```sh
  sudo yum install -y npm
  ```
- 安装 Express.js：
  ```sh
  npm init -y
  npm install express --save
  ```

### 2. 创建项目文件夹NetMonitor，编辑项目文件，并启动服务
- 编辑文件server.js：
  ```
  const express = require('express');
  const bodyParser = require('body-parser');
  const fs = require('fs');

  const app = express();
  app.use(bodyParser.urlencoded({ extended: true }));
  app.use(express.static('public'));

  app.use(bodyParser.urlencoded({ extended: true }));

  app.get('/', (req, res) => {
    res.sendFile(__dirname + '/config.html');
  });

  app.post('/submit', (req, res) => {
    const taskName = req.body.taskName.replace(/ /g, '_');
    const taskDescription = req.body.taskDescription;
    const taskType = req.body.taskType;
    const testMembers = req.body.testMembers;
    const taskStartTime = req.body.taskStartTime;
    const taskEndTime = req.body.taskEndTime;
    const measurementPeriod = req.body.measurementPeriod;
    const measurementPeriodUnit = req.body.measurementPeriodUnit;

    // Convert testMembers to an array
    const testMembersArray = Array.isArray(testMembers) ? testMembers : [testMembers];

    const task = {
      taskName,
      taskDescription,
      taskType,
      testMembers: testMembersArray,
      taskStartTime,
      taskEndTime,
      measurementPeriod,
      measurementPeriodUnit
    };

    const jsonContent = JSON.stringify(task, null, 2);

    const fileName = `${taskName}.json`;

    fs.writeFile(`public/${fileName}`, jsonContent, 'utf8', (err) => {
      if (err) {
        console.error(err);
        res.status(500).send('Error writing JSON file');
      } else {
        console.log(`JSON file ${fileName} has been saved`);
        res.send('Task configuration has been saved');
      }
    });
  });

  app.listen(3000, () => {
    console.log('Server is running on port 3000');
  });

  ```
- 编辑config.html文件：
  ```
  <!DOCTYPE html>
  <html>
  <head>
    <title>配置测量任务</title>
    <script>
      function addTestMember() {
        var input = document.createElement("input");
        input.type = "text";
        input.name = "testMembers";
        input.required = true;
        input.placeholder = "输入成员名称";
        var br = document.createElement("br");
        var container = document.getElementById("testMembersContainer");
        container.appendChild(input);
        container.appendChild(br);
      }
    </script>
  </head>
  <body>
    <h1>配置测量任务</h1>
    <form action="/submit" method="post">
      <label for="taskName">任务名称：</label>
      <input type="text" id="taskName" name="taskName" required><br><br>

      <label for="taskDescription">任务描述：</label><br>
      <textarea id="taskDescription" name="taskDescription" required></textarea><br><br>

      <label for="taskType">任务类型：</label>
      <select id="taskType" name="taskType" required>
        <option value="Basic">Basic</option>
        <option value="AB">AB</option>
      </select><br><br>

      <label for="testMembers">测试成员：</label>
      <div id="testMembersContainer">
        <input type="text" name="testMembers" required placeholder="输入成员名称">
        <br>
      </div>
      <button type="button" onclick="addTestMember()">添加成员</button>
      <br><br>

      <label for="taskStartTime">开始时间：</label>
      <input type="datetime-local" id="taskStartTime" name="taskStartTime" required><br><br>

      <label for="taskEndTime">结束时间：</label>
      <input type="datetime-local" id="taskEndTime" name="taskEndTime" required><br><br>

      <label for="measurementPeriod">测量周期：</label>
      <input type="number" id="measurementPeriod" name="measurementPeriod" required>
      <select id="measurementPeriodUnit" name="measurementPeriodUnit" required>
        <option value="seconds">秒</option>
        <option value="minutes">分钟</option>
        <option value="hours">小时</option>
      </select>
      <br><br>

      <input type="submit" value="保存任务配置"><br><br>
      <input type="button" value="监控结果" onclick="window.location.href='http://[2001:da8:ff:212::13:1]:9090/graph?g0.expr=netmonitor&g0.tab=0&g0.stacked=0&g0.show_exemplars=0&g0.range_input=1h';">
    </form>
  </body>
  </html>
  ```
- 如有防火墙，需开放服务端口：
  ```sh
  sudo firewall-cmd --add-port=3000/tcp --permanent
  sudo firewall-cmd --reload
  ```
- 启动服务：
  ```sh
  node server.js
  ```


### 3. 访问http:[YourIP]:3000
- 通过填写表单并保存提交可生成测量任务，任务文件将保存在public目录下，后台会自动执行该任务。

### 4. 任务目录监控与任务解析下发
- 使用python中的watchdog进行任务文件监控，首先安装：
  ```sh
  pip install watchdog
  ```
- 通过python脚本监控任务并下发：
  ```python
  import time
  from watchdog.observers import Observer
  from watchdog.events import FileSystemEventHandler
  import socket

  monitor_url = "http://[2001:da8:ff:212::13:1]:3000/"

  def connect_to_server(HOST, PORT, file_url):
      # 创建 TCP 套接字
      client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

      try:
          # 连接服务器
          client_socket.settimeout(10)
          client_socket.connect((HOST, PORT))
          print(f'Connected to server: {HOST}:{PORT}')
          
          # 发送文件 URL 给服务器
          file_ip = file_url + '+' + HOST
          print(file_ip)
          client_socket.send(file_ip.encode('utf-8'))

          # 接收服务器响应
          response = client_socket.recv(1024).decode('utf-8')
          print(f'Server response: {response}')

      except:
          print('Failed to connect to the server')

      finally:
          # 关闭套接字连接
          client_socket.close()

  # 定义处理文件事件的处理器类
  class FileHandler(FileSystemEventHandler):
      def on_created(self, event):
          # 处理新文件的逻辑
          print(f"New file created: {event.src_path}")
          
          # 解析文件内容
          with open(event.src_path, 'r') as file:
              file_name = event.src_path.split('/')[-1]
              print(file_name)
              file_content = file.read()
              task_content = eval(file_content)
              # print(file_content)
              testMembers = task_content["testMembers"]
              for member in testMembers:
                  connect_to_server(member, 5555, monitor_url+file_name)
                  time.sleep(7)


  # 定义要监控的目录路径
  directory_to_watch = './public'

  # 创建观察者对象和事件处理器对象
  event_handler = FileHandler()
  observer = Observer()

  # 将事件处理器绑定到观察者，并设置要监控的目录路径
  observer.schedule(event_handler, directory_to_watch, recursive=False)

  # 启动观察者
  observer.start()

  try:
      # 持续监控目录变化
      while True:
          time.sleep(1)
  except KeyboardInterrupt:
      # 用户按下 Ctrl+C 时停止观察者
      observer.stop()

  # 等待观察者线程结束
  observer.join()
  ```

### 5. 安装prometheus与pushgateway，详见文档grafana-and-prometheus。

### 6. 测量节点接收并执行任务
- 安装客户端python包：
  ```sh
  pip install prometheus_client==0.9.0
  ```

- 测量节点有一个测量代理程序mAgent.py，负责接收任务：
  ```python
  import socket
  import subprocess
  import requests

  # TCP 服务器配置
  HOST = '0.0.0.0'
  PORT = 5555

  def parse_file_content(file_url, target_ip):
      response = requests.get(file_url)
      if response.status_code == 200:
          file_content = response.text
          # 在这里执行你的文件内容解析逻辑
          # 例如，打印文件内容
          print(file_content)
          job_name = file_url.split('/')[-1].split('.')[0]
          subprocess.Popen(['python3', 'jobhandler.py', file_content, target_ip, job_name])
          return 'File content parsed successfully'
      else:
          return 'Failed to download file'

  def handle_client(client_socket):
      # 接收客户端数据
      data = client_socket.recv(1024).decode('utf-8')
      if data:
          # 解析数据
          file_ip = data.strip()
          file_url, target_ip = file_ip.split('+')

          # 解析文件内容
          response = parse_file_content(file_url, target_ip)

          # 发送响应给客户端
          client_socket.send(response.encode('utf-8'))

      # 关闭客户端连接
      client_socket.close()

  def start_server():
      # 创建 TCP 套接字
      server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

      # 绑定服务器地址和端口
      server_socket.bind((HOST, PORT))

      # 开始监听连接
      server_socket.listen()

      print(f'TCP server is running on {HOST}:{PORT}')

      while True:
          # 等待客户端连接
          client_socket, client_address = server_socket.accept()
          print(f'Client connected: {client_address[0]}:{client_address[1]}')

          # 处理客户端请求
          handle_client(client_socket)

  if __name__ == '__main__':
      start_server()
  ```

- 同样，需要事先开启服务端口：
  ```sh
  sudo firewall-cmd --add-port=5555/tcp --permanent
  sudo firewall-cmd --reload
  ```
- 执行测量任务并上传测量结果：jobhandler.py
  ```python
  import sys
  import time
  import subprocess
  import re
  from prometheus_client import CollectorRegistry, Gauge, push_to_gateway

  registry=CollectorRegistry()
  g = Gauge('netmonitor', 'netmonitor gauge', ['sip', 'dip', 'metric'], registry=registry)
  pushgateway_url = '[2001:da8:ff:212::13:1]:9091'

  def upload(data, job_name):
      # 设置指标值
      g.labels(sip=data["sip"], dip=data["dip"], metric=data["metric"]).set(data[data['metric']])

      # 推送数据至Pushgateway
      push_to_gateway(pushgateway_url, job=job_name, registry=registry)
      #time.sleep(0.1)

  def run_perftrace_basic(svr_ip):
      # 构造perftrace命令
      command = ['./PerfTrace/perftrace_cli','-c','10','-i','10ms','-m','1','-s',svr_ip]  # 这里使用了Linux/Mac的perftrace命令参数，Windows下可以修改为相应的参数

      # 执行perftrace命令
      process = subprocess.Popen(command, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
      output, error = process.communicate()

      # 等待命令执行完成
      process.wait()

      # 检查返回码，非零值表示命令执行失败
      if process.returncode != 0:
          print(f"Error executing perftrace command: {error.decode('utf-8')}")
          return

      # 输出perftrace命令的结果
      return output.decode('utf-8')

  def run_perftrace_AB(svr_ip):
      # 构造perftrace命令
      command = ['./PerfTrace/perftrace_cli','-m','2','-s',svr_ip]  # 这里使用了Linux/Mac的perftrace命令参数，Windows下可以修改为相应的参数

      # 执行perftrace命令
      process = subprocess.Popen(command, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
      output, error = process.communicate()

      # 等待命令执行完成
      process.wait()

      # 检查返回码，非零值表示命令执行失败
      if process.returncode != 0:
          print(f"Error executing perftrace command: {error.decode('utf-8')}")
          return

      # 输出perftrace命令的结果
      return output.decode('utf-8')


  def parse_basic_result(result_str, sip, dip, job_name):
      # 提取Two way delay的min/aver/max值
      two_way_delay = re.findall(r'Two way delay:\s+min/aver/max = ([\d.]+)/([\d.]+)/([\d.]+)', result_str)
      if two_way_delay:
          two_way_delay_aver = float(two_way_delay[0][1])
      else:
          two_way_delay_aver = None

      # 提取One way delay的min/aver/max值
      one_way_delay = re.findall(r'One way delay:\s+Source->Dest: min/aver/max = ([\d.]+)/([\d.]+)/([\d.]+)', result_str)
      if one_way_delay:
          one_way_delay_sd_aver = float(one_way_delay[0][1])
      else:
          one_way_delay_sd_aver = None

      one_way_delay_ds = re.findall(r'Dest->Source: min/aver/max = ([\d.-]+)/([\d.-]+)/([\d.-]+)', result_str)
      if one_way_delay_ds:
          one_way_delay_ds_aver = float(one_way_delay_ds[0][1])
      else:
          one_way_delay_ds_aver = None

      # 提取Two way jitter值
      two_way_jitter = re.findall(r'Two way jitter:\s+([\d.]+)', result_str)
      if two_way_jitter:
          two_way_jitter = float(two_way_jitter[0])
      else:
          two_way_jitter = None

      # 提取One way jitter值
      one_way_jitter = re.findall(r'One way jitter:\s+Source->Dest: ([\d.]+)', result_str)
      if one_way_jitter:
          one_way_jitter_sd = float(one_way_jitter[0])
      else:
          one_way_jitter_sd = None

      one_way_jitter_ds = re.findall(r'Dest->Source: ([\d.]+)', result_str)
      if one_way_jitter_ds:
          one_way_jitter_ds = float(one_way_jitter_ds[0])
      else:
          one_way_jitter_ds = None

      # 提取Two way packet loss rate值
      two_way_loss_rate = re.findall(r'Two way packet loss rate:\s+([\d.]+)%', result_str)
      if two_way_loss_rate:
          two_way_loss_rate = float(two_way_loss_rate[0])/100
      else:
          two_way_loss_rate = None

      # 提取One way packet loss rate值
      one_way_loss_rate = re.findall(r'One way packet loss rate:\s+Source->Dest: ([\d.]+)%', result_str)
      if one_way_loss_rate:
          one_way_loss_rate_sd = float(one_way_loss_rate[0])/100
      else:
          one_way_loss_rate_sd = None

      one_way_loss_rate_ds = re.findall(r'Dest->Source: ([\d.]+)%', result_str)
      if one_way_loss_rate_ds:
          one_way_loss_rate_ds = float(one_way_loss_rate_ds[0])/100
      else:
          one_way_loss_rate_ds = None
      print(two_way_delay_aver, one_way_delay_sd_aver, one_way_delay_ds_aver, two_way_jitter, one_way_jitter_sd, one_way_jitter_ds, two_way_loss_rate, one_way_loss_rate_sd, one_way_loss_rate_ds)
      upload({'sip':sip, 'dip':dip, 'RTT': two_way_delay_aver,'metric': 'RTT'}, job_name)
      upload({'sip':sip, 'dip':dip, 'OWD': one_way_delay_sd_aver,'metric': 'OWD'}, job_name)
      upload({'sip':dip, 'dip':sip, 'OWD': one_way_delay_ds_aver,'metric': 'OWD'}, job_name)
      upload({'sip':sip, 'dip':dip, 'OW_Jitter': one_way_jitter_sd,'metric': 'OW_Jitter'}, job_name)
      upload({'sip':dip, 'dip':sip, 'OW_Jitter': one_way_jitter_ds,'metric': 'OW_Jitter'}, job_name)
      upload({'sip':sip, 'dip':dip, 'RT_Loss': two_way_loss_rate,'metric': 'RT_Loss'}, job_name)
      upload({'sip':sip, 'dip':dip, 'OW_Loss': one_way_loss_rate_sd,'metric': 'OW_Loss'}, job_name)
      upload({'sip':dip, 'dip':sip, 'OW_Loss': one_way_loss_rate_ds,'metric': 'OW_Loss'}, job_name)

  def parse_AB_result(result_str, sip, dip, job_name):
      ab = re.findall(r'final result: ([\d.]+)bps', result_str)
      print(float(ab[0]))
      upload({'sip':sip, 'dip':dip, 'AB': float(ab[0]),'metric': 'AB'}, job_name)


  if __name__ == '__main__':
      task_content = eval(sys.argv[1])
      target_ip = sys.argv[2]
      job_name = sys.argv[3]
      # print(task_content)
      # print(target_ip)
      # print(job_name)
      start_time = task_content['taskStartTime']
      start_ts = int(time.mktime(time.strptime(start_time, "%Y-%m-%dT%H:%M")))
      end_time = task_content['taskEndTime']
      end_ts = int(time.mktime(time.strptime(end_time, "%Y-%m-%dT%H:%M")))
      period = int(task_content['measurementPeriod'])
      if task_content["measurementPeriodUnit"] == "minutes":
          period *= 60
      elif task_content["measurementPeriodUnit"] == "hours":
          period *= 3600
      test_members = sorted(task_content["testMembers"])
      task_type = task_content["taskType"]
      now_ts = int(time.time())
      if end_ts > start_ts and end_ts > now_ts:
          if start_ts > now_ts:
              time.sleep(start_ts - now_ts)
              now_ts = int(time.time())
          while True:
              if task_type == "Basic":
                  # meas_func()
                  for member in test_members:
                      if member != target_ip:
                          result_basic = run_perftrace_basic(member)
                          if result_basic and "INSERT INTO PerfRecords VALUES" in result_basic:
                              parse_basic_result(result_basic,target_ip,member, job_name)
                      else:
                          break
              else:
                  for member in test_members:
                      if member != target_ip:
                          result_ab = run_perftrace_AB(member)
                          if result_ab and "INSERT INTO PerfRecords VALUES" in result_ab:
                              parse_AB_result(result_ab,target_ip,member, job_name)
                      else:
                          continue
              print("OK")
              time.sleep(max(0, now_ts + period - int(time.time())))
              now_ts = int(time.time())
              if now_ts > end_ts:
                  break
  ```

### 7. 在所有测量节点上下载测量工具PerfTrace并运行：
  ```sh
  git clone https://github.com/Yiu-chung/PerfTrace.git
  ./perftrace_srv
  ```
- 开启端口：
  ```sh
  firewall-cmd --permanent --zone=public --add-port=19999/udp
  firewall-cmd --permanent --zone=public --add-port=19999/tcp
  firewall-cmd --reload
  ```