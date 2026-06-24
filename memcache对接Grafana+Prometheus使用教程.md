
当前使用的版本为grafana-13.0.1与prometheus-3.12.0，需要根据使用服务器的架构来选择下载的小版本

下载地址：

grafana：https://grafana.com/grafana/download/13.0.1?platform=arm


prometheus：https://prometheus.io/download/
 
首先，你需要确认你的服务器架构。你可以通过以下命令来查看：
```
uname -a
```
如果输出中包含 aarch64，则你的服务器架构为 ARM64。

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
使用默认的用户名和密码 admin 登录，并设置新密码。


添加Prometheus数据源：


在左侧菜单中选择 Connections -> Data Sources -> Add data source。
选择 Prometheus。
在 HTTP 部分，输入Prometheus的URL，例如 http://141.61.39.235:9090/
点击 Save & Test。

# 4. 创建仪表盘
在Grafana中，选择 Create -> Dashboard。
添加新的面板，选择 Add query，选择你刚刚添加的Prometheus数据源。
选择你想要显示的指标，例如 memcache 相关的指标。







 效果展示图





```
{
  "annotations": [
    {
      "kind": "AnnotationQuery",
      "spec": {
        "builtIn": true,
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "query": {
          "datasource": {
            "name": "-- Grafana --"
          },
          "group": "grafana",
          "kind": "DataQuery",
          "spec": {},
          "version": "v0"
        }
      }
    }
  ],
  "cursorSync": "Off",
  "editable": true,
  "elements": {
    "panel-1": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_alloc_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_alloc_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_alloc_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "Alloc requests, successes, failures",
        "id": 1,
        "links": [],
        "title": "Alloc",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-10": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_query_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_query_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_query_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_query_not_found_total{job=\"prometheus\"}",
                      "legendFormat": "not_found",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "D"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "Query requests, successes, failures, not_found",
        "id": 10,
        "links": [],
        "title": "Query",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-11": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_query_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_query_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_query_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_query_not_found_total{job=\"prometheus\"}",
                      "legendFormat": "not_found",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "D"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "BatchQuery requests, successes, failures, not_found",
        "id": 11,
        "links": [],
        "title": "BatchQuery",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-12": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_get_all_keys_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_get_all_keys_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_get_all_keys_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "GetAllKeys requests, successes, failures",
        "id": 12,
        "links": [],
        "title": "GetAllKeys",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-13": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_exist_key_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_exist_key_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_exist_key_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_exist_key_not_found_total{job=\"prometheus\"}",
                      "legendFormat": "not_found",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "D"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "ExistKey requests, successes, failures, not_found",
        "id": 13,
        "links": [],
        "title": "ExistKey",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-14": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_exist_key_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_exist_key_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_exist_key_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_exist_key_not_found_total{job=\"prometheus\"}",
                      "legendFormat": "not_found",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "D"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "BatchExistKey requests, successes, failures, not_found",
        "id": 14,
        "links": [],
        "title": "BatchExistKey",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-15": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_mount_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_mount_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_mount_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "Mount requests, successes, failures",
        "id": 15,
        "links": [],
        "title": "Mount",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-16": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_unmount_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_unmount_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_unmount_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "Unmount requests, successes, failures",
        "id": 16,
        "links": [],
        "title": "Unmount",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-17": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_evict_operations_total{job=\"prometheus\"}",
                      "legendFormat": "evict_operations",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "Evict operations",
        "id": 17,
        "links": [],
        "title": "Evict",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-18": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_stored_keys{job=\"prometheus\"}",
                      "legendFormat": "stored_keys",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "Number of stored keys",
        "id": 18,
        "links": [],
        "title": "Stored Keys",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-19": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_segment_capacity_bytes{job=\"prometheus\"}",
                      "legendFormat": "{{segment}}",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "Segment total capacity by segment",
        "id": 19,
        "links": [],
        "title": "Segment Capacity (bytes)",
        "vizConfig": {
          "group": "stat",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "colorMode": "background",
              "graphMode": "none",
              "justifyMode": "auto",
              "orientation": "auto",
              "percentChangeColorMode": "standard",
              "reduceOptions": {
                "calcs": [
                  "lastNotNull"
                ],
                "fields": "",
                "values": false
              },
              "showPercentChange": false,
              "textMode": "auto",
              "wideLayout": true
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-2": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_alloc_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_alloc_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_alloc_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "BatchAlloc requests, successes, failures",
        "id": 2,
        "links": [],
        "title": "BatchAlloc",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-20": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_segment_allocated_bytes{job=\"prometheus\"}",
                      "legendFormat": "{{segment}}",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "Segment allocated bytes by segment",
        "id": 20,
        "links": [],
        "title": "Segment Allocated (bytes)",
        "vizConfig": {
          "group": "bargauge",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "displayMode": "lcd",
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": false
              },
              "maxVizHeight": 300,
              "minVizHeight": 16,
              "minVizWidth": 8,
              "namePlacement": "auto",
              "orientation": "horizontal",
              "reduceOptions": {
                "calcs": [
                  "lastNotNull"
                ],
                "fields": "",
                "values": false
              },
              "showUnfilled": true,
              "sizing": "auto",
              "valueMode": "color"
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-21": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_total_capacity_bytes{job=\"prometheus\"}",
                      "legendFormat": "{{medium}}",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "Total capacity by medium",
        "id": 21,
        "links": [],
        "title": "Total Capacity (bytes)",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "bars",
                  "fillOpacity": 100,
                  "gradientMode": "hue",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "normal"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-22": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_allocated_bytes{job=\"prometheus\"}",
                      "legendFormat": "{{medium}}",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "Allocated bytes by medium",
        "id": 22,
        "links": [],
        "title": "Allocated Bytes",
        "vizConfig": {
          "group": "stat",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "colorMode": "background",
              "graphMode": "none",
              "justifyMode": "auto",
              "orientation": "auto",
              "percentChangeColorMode": "standard",
              "reduceOptions": {
                "calcs": [
                  "lastNotNull"
                ],
                "fields": "",
                "values": false
              },
              "showPercentChange": false,
              "textMode": "auto",
              "wideLayout": true
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-3": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_get_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_get_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_get_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_get_not_found_total{job=\"prometheus\"}",
                      "legendFormat": "not_found",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "D"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "Get requests, successes, failures, not_found",
        "id": 3,
        "links": [],
        "title": "Get",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-4": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_get_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_get_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_get_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_get_not_found_total{job=\"prometheus\"}",
                      "legendFormat": "not_found",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "D"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "BatchGet requests, successes, failures, not_found",
        "id": 4,
        "links": [],
        "title": "BatchGet",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-5": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_remove_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_remove_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_remove_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_remove_not_found_total{job=\"prometheus\"}",
                      "legendFormat": "not_found",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "D"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "Remove requests, successes, failures, not_found",
        "id": 5,
        "links": [],
        "title": "Remove",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-6": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_remove_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_remove_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_remove_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_remove_not_found_total{job=\"prometheus\"}",
                      "legendFormat": "not_found",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "D"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "BatchRemove requests, successes, failures, not_found",
        "id": 6,
        "links": [],
        "title": "BatchRemove",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-7": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_remove_all_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_remove_all_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_remove_all_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "RemoveAll requests, successes, failures",
        "id": 7,
        "links": [],
        "title": "RemoveAll",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-8": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_update_state_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_update_state_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_update_state_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_update_state_not_found_total{job=\"prometheus\"}",
                      "legendFormat": "not_found",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "D"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "UpdateState requests, successes, failures, not_found",
        "id": 8,
        "links": [],
        "title": "UpdateState",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    },
    "panel-9": {
      "kind": "Panel",
      "spec": {
        "data": {
          "kind": "QueryGroup",
          "spec": {
            "queries": [
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_update_state_requests_total{job=\"prometheus\"}",
                      "legendFormat": "requests",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "A"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_update_state_successes_total{job=\"prometheus\"}",
                      "legendFormat": "successes",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "B"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_update_state_failures_total{job=\"prometheus\"}",
                      "legendFormat": "failures",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "C"
                }
              },
              {
                "kind": "PanelQuery",
                "spec": {
                  "hidden": false,
                  "query": {
                    "datasource": {
                      "name": "afq04xw0hiccgf"
                    },
                    "group": "prometheus",
                    "kind": "DataQuery",
                    "spec": {
                      "editorMode": "builder",
                      "expr": "memcache_batch_update_state_not_found_total{job=\"prometheus\"}",
                      "legendFormat": "not_found",
                      "range": true
                    },
                    "version": "v0"
                  },
                  "refId": "D"
                }
              }
            ],
            "queryOptions": {},
            "transformations": []
          }
        },
        "description": "BatchUpdateState requests, successes, failures, not_found",
        "id": 9,
        "links": [],
        "title": "BatchUpdateState",
        "vizConfig": {
          "group": "timeseries",
          "kind": "VizConfig",
          "spec": {
            "fieldConfig": {
              "defaults": {
                "color": {
                  "mode": "palette-classic"
                },
                "custom": {
                  "axisBorderShow": false,
                  "axisCenteredZero": false,
                  "axisColorMode": "text",
                  "axisLabel": "",
                  "axisPlacement": "auto",
                  "barAlignment": 0,
                  "barWidthFactor": 0.6,
                  "drawStyle": "line",
                  "fillOpacity": 0,
                  "gradientMode": "none",
                  "hideFrom": {
                    "legend": false,
                    "tooltip": false,
                    "viz": false
                  },
                  "insertNulls": false,
                  "lineInterpolation": "linear",
                  "lineWidth": 1,
                  "pointSize": 5,
                  "scaleDistribution": {
                    "type": "linear"
                  },
                  "showPoints": "auto",
                  "showValues": false,
                  "spanNulls": false,
                  "stacking": {
                    "group": "A",
                    "mode": "none"
                  },
                  "thresholdsStyle": {
                    "mode": "off"
                  }
                },
                "thresholds": {
                  "mode": "absolute",
                  "steps": [
                    {
                      "color": "green",
                      "value": 0
                    },
                    {
                      "color": "red",
                      "value": 80
                    }
                  ]
                }
              },
              "overrides": []
            },
            "options": {
              "annotations": {
                "clustering": -1,
                "multiLane": false
              },
              "legend": {
                "calcs": [],
                "displayMode": "list",
                "placement": "bottom",
                "showLegend": true
              },
              "tooltip": {
                "hideZeros": false,
                "mode": "single",
                "sort": "none"
              }
            }
          },
          "version": "13.0.1"
        }
      }
    }
  },
  "layout": {
    "kind": "GridLayout",
    "spec": {
      "items": [
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-1"
            },
            "height": 8,
            "width": 12,
            "x": 0,
            "y": 0
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-2"
            },
            "height": 8,
            "width": 12,
            "x": 12,
            "y": 0
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-3"
            },
            "height": 8,
            "width": 12,
            "x": 0,
            "y": 8
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-4"
            },
            "height": 8,
            "width": 12,
            "x": 12,
            "y": 8
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-5"
            },
            "height": 8,
            "width": 12,
            "x": 0,
            "y": 16
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-6"
            },
            "height": 8,
            "width": 12,
            "x": 12,
            "y": 16
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-7"
            },
            "height": 8,
            "width": 12,
            "x": 0,
            "y": 24
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-8"
            },
            "height": 8,
            "width": 12,
            "x": 12,
            "y": 24
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-9"
            },
            "height": 8,
            "width": 12,
            "x": 0,
            "y": 32
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-10"
            },
            "height": 8,
            "width": 12,
            "x": 12,
            "y": 32
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-11"
            },
            "height": 8,
            "width": 12,
            "x": 0,
            "y": 40
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-12"
            },
            "height": 8,
            "width": 12,
            "x": 12,
            "y": 40
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-13"
            },
            "height": 8,
            "width": 12,
            "x": 0,
            "y": 48
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-14"
            },
            "height": 8,
            "width": 12,
            "x": 12,
            "y": 48
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-15"
            },
            "height": 8,
            "width": 12,
            "x": 0,
            "y": 56
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-16"
            },
            "height": 8,
            "width": 12,
            "x": 12,
            "y": 56
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-17"
            },
            "height": 8,
            "width": 12,
            "x": 0,
            "y": 64
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-18"
            },
            "height": 8,
            "width": 12,
            "x": 12,
            "y": 64
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-19"
            },
            "height": 8,
            "width": 12,
            "x": 0,
            "y": 72
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-20"
            },
            "height": 8,
            "width": 12,
            "x": 12,
            "y": 72
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-21"
            },
            "height": 8,
            "width": 12,
            "x": 0,
            "y": 80
          }
        },
        {
          "kind": "GridLayoutItem",
          "spec": {
            "element": {
              "kind": "ElementReference",
              "name": "panel-22"
            },
            "height": 8,
            "width": 12,
            "x": 12,
            "y": 80
          }
        }
      ]
    }
  },
  "links": [],
  "liveNow": false,
  "preferences": {
    "layout": {
      "kind": "GridLayout",
      "spec": {
        "items": []
      }
    }
  },
  "preload": false,
  "tags": [
    "memcache"
  ],
  "timeSettings": {
    "autoRefresh": "",
    "autoRefreshIntervals": [
      "5s",
      "10s",
      "30s",
      "1m",
      "5m",
      "15m",
      "30m",
      "1h",
      "2h",
      "1d"
    ],
    "fiscalYearStartMonth": 0,
    "from": "now-6h",
    "hideTimepicker": false,
    "timezone": "browser",
    "to": "now"
  },
  "title": "Memcache Dashboard",
  "variables": []
}
```



 
 
 
 
 
 
 
 
 
 
 
 
 