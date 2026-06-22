当前使用的版本为grafana-13.0.1与prometheus-3.12.0，需要根据使用服务器的架构来选择下载的小版本，下载地址：
grafana：https://prometheus.io/download/
prometheus：https://grafana.com/grafana/download/13.0.1?platform=arm
 
可以通过在环境uname -a查看使用的架构
 
一、文件下载解压
 
1.Grafana
打开上面的官网下载地址之后通过unma -a查看到的环境信息来选择二进制文件
 
aarch64下载参考
wget https://dl.grafana.com/grafana-enterprise/release/13.0.1/grafana-enterprise_13.0.1_24542347077_linux_arm64.tar.gz --no-check-certificate
tar -zxvf grafana-enterprise_13.0.1_24542347077_linux_arm64.tar.gz

需要注意的是使用官网的下载指令时可能会提示你在最后加上--no-check-certificate 
如果在下载时架构选择错误或文件不完整均会在解压的最后有报错提示
 
 
2.prometheus
同理，也是在官网下载地址根据自己的服务器架构选择不同的操作系统版本，prometheus可以通过直接下载到本在再通过xftp传到环境上
 
aarch64下载参考（本地下载xftp到环境上解压即可）
 `tar -zxvf prometheus-3.12.0.linux-amd64.tar.gz`
 
 
二、启动服务
 
1、prometheus
启动prometheus之前需要修改一下prometheus.yml文件增加监听的端口
文件地址：/prometheus-3.12.0.linux-arm64/ prometheus.yml   
示例：

```
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    monitor: 'codelab-monitor'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'qwen3-32b'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['141.61.39.235:5000']
    # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'qwen3-32b-1'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['141.61.39.235:6000']
  - job_name: 'memcache'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['141.61.39.235:8000'] 
```
只需加上监听端口（可多个）与job name
进入解压完的prometheus-3.12.0.linux-arm64文件夹输入
 `./prometheus --config.file=prometheus.yml`
 
2.grafana
进入解压完的grafana-13.0.1文件夹输入
 `./bin/grafana server`
 
 
在启动完成后grafana与prometheus之后可以在网页端打开 环境ip：3000进入grafana看板
例如我在141.61.39.235则输入
http://141.61.39.235:3000/
在第一次登陆时会有一个账号密码默认都是admin，在输完第一遍时会让你更新新密码，建议继续使用admin
 
 
 
三、看板配置
首先需要在grafana看板连接prometheus数据源
在左侧的connections里选择add new connection,add new connection
选择prometheus
进行端口设置，一般prometheus端口为9090，则输入http://141.61.39.235（服务器ip）:9090/
最底下选择save&test
连接成功之后可以在data sources里创建仪表盘
连接成功后就可以在new panel里面看到带memcache的关键字的数据了
grafana原始数据来自141.61.39.235:8000/metrics ，可以用自己的服务器ip打开查看所有的数据


![image.png](https://raw.gitcode.com/user-images/assets/7672915/1b51b1b4-0884-438c-b571-da15e62f2767/image.png 'image.png')
 
 ![image.png](https://raw.gitcode.com/user-images/assets/7672915/12ab887a-71c4-4bd6-8fd4-ad3772bef8ee/image.png 'image.png')
 
 
 ![image.png](https://raw.gitcode.com/user-images/assets/7672915/b192dd1e-6154-4bb8-90f5-9c28bdb0b307/image.png 'image.png')
 
 
 
 
 
 
 
 
 
 
 
 
 