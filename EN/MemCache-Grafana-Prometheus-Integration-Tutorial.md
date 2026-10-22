
This guide uses Grafana 10.4.2 and Prometheus 3.12.0. Choose the download package that matches your server architecture.



Download links:



grafana：https://grafana.com/grafana/download/10.4.2?platform=arm



prometheus：https://prometheus.io/download/


 
First, check your server architecture with:


```
uname -a
```



# 1. Download and Extract the Packages


Grafana


Download the Grafana package for your server architecture:


```
wget https://dl.grafana.com/enterprise/release/grafana-enterprise-10.4.2.linux-arm64.tar.gz --no-check-certificate
tar -zxvf grafana-enterprise-10.4.2.linux-arm64.tar.gz
```


Prometheus
Likewise, download the Prometheus package for your server architecture:


```
wget https://github.com/prometheus/prometheus/releases/download/v3.12.0/prometheus-3.12.0.linux-arm64.tar.gz
tar -zxvf prometheus-3.12.0.linux-arm64.tar.gz
```


# 2. Start the Services
Prometheus

Edit `prometheus.yml` to configure the scrape target and port:



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



Start Prometheus:



```
cd prometheus-3.12.0.linux-arm64
./prometheus --config.file=prometheus.yml
```





Start Grafana:



```
cd grafana-v10.4.2/
./bin/grafana server
```



# 3. Configure Grafana



Open Grafana's web interface in a browser:



http://141.61.39.235:3000/


Sign in with the default username and password, `admin`, and set a new password.



Add Prometheus as a data source:



In the left menu, select **Connections → Data Sources → Add data source**.


Select **Prometheus**.


In the **HTTP** section, enter the Prometheus URL, for example `http://141.61.39.235:9090/`.


Click **Save & Test**.






# 4. Create a Dashboard


In Grafana, select **Dashboard → New**.


Add a panel, select **Add query**, and choose the Prometheus data source you just added.



Choose the metrics to display, such as MemCache metrics.


You can also select **Import** after choosing **Dashboard → New** to use an existing dashboard template.



Template 1:


```
https://gitcode.com/Ascend/memcache/blob/master/example/metrics/memcache_dashboard_graph.json
```

Example dashboard:


![1FF81AF5-30BB-4E19-8BAA-7EE50CE9DBE7.jpg](https://raw.gitcode.com/user-images/assets/7672915/d6e337f2-54ca-4e83-87d4-07b4b377ba8f/1FF81AF5-30BB-4E19-8BAA-7EE50CE9DBE7.jpg '1FF81AF5-30BB-4E19-8BAA-7EE50CE9DBE7.jpg')




Template 2:

```
https://gitcode.com/Ascend/memcache/blob/master/example/metrics/memcache_dashboard_math.json
```

Example dashboard:
 
 ![A3A78D98-4979-45FE-F22A-CF03D292611F.jpg](https://raw.gitcode.com/user-images/assets/7672915/5df55eb2-bec4-45a0-a71d-cb34a708f9c7/A3A78D98-4979-45FE-F22A-CF03D292611F.jpg 'A3A78D98-4979-45FE-F22A-CF03D292611F.jpg')

