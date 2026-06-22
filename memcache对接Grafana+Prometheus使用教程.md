当前使用的版本为grafana-13.0.1与prometheus-3.12.0，需要根据使用服务器的架构来选择下载的小版本，下载地址：
grafana：https://prometheus.io/download/
prometheus：https://grafana.com/grafana/download/13.0.1?platform=arm
 
首先，你需要确认你的服务器架构。你可以通过以下命令来查看：

uname -a
如果输出中包含 aarch64，则你的服务器架构为 ARM64。

2. 下载和解压文件
Grafana
根据你的服务器架构，下载相应的Grafana版本：

wget https://dl.grafana.com/grafana-enterprise/release/13.0.1/grafana-enterprise_13.0.1_24542347077_linux_arm64.tar.gz --no-check-certificate
tar -zxvf grafana-enterprise_13.0.1_24542347077_linux_arm64.tar.gz
Prometheus
同样，根据你的服务器架构，下载相应的Prometheus版本：

wget https://github.com/prometheus/prometheus/releases/download/v3.12.0/prometheus-3.12.0.linux-arm64.tar.gz
tar -zxvf prometheus-3.12.0.linux-arm64.tar.gz
3. 启动服务
Prometheus
修改 prometheus.yml 文件，增加监听的端口和目标：

global:
  scrape_interval: 15s
  external_labels:
    monitor: 'codelab-monitor'

scrape_configs:
  - job_name: 'qwen3-32b'
    scrape_interval: 5s
    static_configs:
      - targets: ['141.61.39.235:5000']
  - job_name: 'qwen3-32b-1'
    scrape_interval: 5s
    static_configs:
      - targets: ['141.61.39.235:6000']
  - job_name: 'memcache'
    scrape_interval: 5s
    static_configs:
      - targets: ['141.61.39.235:8000']
启动Prometheus：

cd prometheus-3.12.0.linux-arm64
./prometheus --config.file=prometheus.yml
Grafana
启动Grafana：
cd grafana-13.0.1
./bin/grafana-server
4. 配置Grafana
打开浏览器，访问Grafana的Web界面：

http://141.61.39.235:3000/
使用默认的用户名和密码 admin 登录，并设置新密码。

添加Prometheus数据源：

在左侧菜单中选择 Connections -> Data Sources -> Add data source。
选择 Prometheus。
在 HTTP 部分，输入Prometheus的URL，例如 http://141.61.39.235:9090/。
点击 Save & Test。
5. 创建仪表盘
在Grafana中，选择 Create -> Dashboard。
添加新的面板，选择 Add query，选择你刚刚添加的Prometheus数据源。
选择你想要显示的指标，例如 memcache 相关的指标。
可能的问题及解决方法
架构不匹配：

确认你下载的Grafana和Prometheus版本与你的服务器架构匹配。如果架构不匹配，解压时会报错。
文件完整性：

确认下载的文件完整无损。如果文件不完整，解压时会报错。
配置文件错误：

确认 prometheus.yml 文件的格式正确，没有语法错误。
端口冲突：

确认Prometheus和Grafana使用的端口没有被其他服务占用。
防火墙和网络问题：

确认防火墙没有阻止Prometheus和Grafana的端口（9090和3000）。
如果你在执行上述步骤时遇到具体错误，请提供错误信息，以便进一步诊断问题。


![image.png](https://raw.gitcode.com/user-images/assets/7672915/1b51b1b4-0884-438c-b571-da15e62f2767/image.png 'image.png')
 
 ![image.png](https://raw.gitcode.com/user-images/assets/7672915/12ab887a-71c4-4bd6-8fd4-ad3772bef8ee/image.png 'image.png')
 
 
 ![image.png](https://raw.gitcode.com/user-images/assets/7672915/b192dd1e-6154-4bb8-90f5-9c28bdb0b307/image.png 'image.png')
 
 
 
 
 
 
 
 
 
 
 
 
 