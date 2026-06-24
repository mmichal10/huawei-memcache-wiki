
当前使用的版本为grafana-13.0.1与prometheus-3.12.0，需要根据使用服务器的架构来选择下载的小版本



下载地址：



grafana：https://grafana.com/grafana/download/13.0.1?platform=arm



prometheus：https://prometheus.io/download/


 
首先，你需要确认你的服务器架构。你可以通过以下命令来查看：


```
uname -a
```



# 1. 下载和解压文件


Grafana


根据你的服务器架构，下载相应的Grafana版本：


```
wget https://dl.grafana.com/grafana-enterprise/release/13.0.1/grafana-enterprise_13.0.1_24542347077_linux_arm64.tar.gz --no-check-certificate
tar -zxvf grafana-enterprise_13.0.1_24542347077_linux_arm64.tar.gz
```


Prometheus
同样，根据你的服务器架构，下载相应的Prometheus版本：


```
wget https://github.com/prometheus/prometheus/releases/download/v3.12.0/prometheus-3.12.0.linux-arm64.tar.gz
tar -zxvf prometheus-3.12.0.linux-arm64.tar.gz
```


# 2. 启动服务
Prometheus

修改 prometheus.yml 文件，增加监听的端口和目标：



```
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["141.61.39.235:8000"]
       # The label name is added as a label `label_name=<label_value>` to any timeseries scraped from this config.
        labels:
          app: "prometheus"


```



启动Prometheus：



```
cd prometheus-3.12.0.linux-arm64
./prometheus --config.file=prometheus.yml
```





启动Grafana：



```
cd grafana-13.0.1
./bin/grafana server
```



# 3. 配置Grafana



打开浏览器，访问Grafana的Web界面：



http://141.61.39.235:3000/


使用默认的用户名和密码 admin 登录，并设置新密码



添加Prometheus数据源：



在左侧菜单中选择 Connections -> Data Sources -> Add data source


选择 Prometheus


在 HTTP 部分，输入Prometheus的URL，例如 http://141.61.39.235:9090/


点击 Save & Test




# 4. 创建仪表盘


在Grafana中，选择 Create -> Dashboard


添加新的面板，选择 Add query，选择你刚刚添加的Prometheus数据源



选择你想要显示的指标，例如 memcache 相关的指标。




示例展示图json


`https://gitcode.com/yuandongsheng/yds_mmc/blob/main/memcache_dashboard.json`

 效果展示图


![1872BAB6-C1FB-465D-CD26-E5A972254234.jpg](https://raw.gitcode.com/user-images/assets/7672915/1ef67ea8-66b6-4775-ae06-c8471ed9ec73/1872BAB6-C1FB-465D-CD26-E5A972254234.jpg '1872BAB6-C1FB-465D-CD26-E5A972254234.jpg')

