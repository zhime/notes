### **1、简介**

Prometheus 是一个开源的监控和告警系统，它专注于时序数据的采集和存储。其主要特点包括：

1. 时序数据存储：Prometheus 会定期从各种数据源拉取数据（如主机的资源使用情况等），并将其以时间序列的形式存储。
2. 查询语言：Prometheus 使用 PromQL（Prometheus Query Language）来查询和聚合数据，能够灵活地提取和分析时序数据。
3. 告警机制：Prometheus 提供了告警功能，可以设置阈值条件，触发告警并通过外部服务（如邮件、钉钉）进行通知。
4. 多维度数据采集：支持多种指标类型，如 counter、gauge、histogram 和 summary。
5. 自动发现：Prometheus 可以与多种云平台和容器化平台（如 Kubernetes）集成，自动发现监控目标

Grafana 是一个开源的数据可视化和分析平台，广泛用于与 Prometheus 等监控工具结合使用，提供丰富的图形化展示功能。其特点包括：

1. 数据源支持：Grafana 可以与多个数据源进行集成，包括 Prometheus、InfluxDB、MySQL、Elasticsearch 等。
2. 丰富的可视化面板：Grafana 提供了多种图表和仪表盘（例如时间序列图、柱状图、饼图等），可以自由定制视图，以直观的方式展示监控数据。
3. 告警功能：Grafana 支持基于数据源的查询结果设置告警规则，告警可以通过多种渠道（如电子邮件、Slack、Webhook）发送。
4. 仪表盘共享与管理：Grafana 允许创建和共享自定义仪表盘，方便团队协作和数据共享

### **2、安装**

#### **下载地址**

`Prometheus`：[https://prometheus.io/download](https://prometheus.io/download/)

`Grafana`：https://grafana.com/grafana/download?pg=oss-graf&plcmt=hero-btn-1

`prometheus-webhook-dingtalk`：https://github.com/timonwong/prometheus-webhook-dingtalk/releases

#### **软件包**

需下载软件包：`Prometheus`、`AlertManager`、`Grfana`、`node_exporter`、`prometheus-webhook-dingtalk`

![image](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image.png)

#### **安装**

将下载好的`tar`包统一解压至`/opt/prometheus`

![image-1](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-1.png)

##### **Prometheus**

`systemd`服务文件：

```
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
Restart=on-failure
ExecStart=/opt/prometheus/prometheus-2.37.8/prometheus \
  --config.file=/opt/prometheus/prometheus-2.37.8/prometheus.yml \
  --storage.tsdb.path=/opt/prometheus/prometheus-2.37.8/data \
  --storage.tsdb.retention.time=60d \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

配置文件：

```
global:
  scrape_interval: 30s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 30s # Evaluate rules every 15 seconds. The default is every 1 minute.

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - 192.168.0.47:9093

rule_files:
  - "rules/*.yml"

scrape_configs:
  - job_name: "node-exporter"
    file_sd_configs:
      - files:
        - targets/node-exporter/*.yml
        refresh_interval: 10m
```

目标配置：

```
- targets:
  - "192.168.0.47:9100"
  labels:
    hostname: prometheus
```

规则配置：

```
groups:
- name: Hosts.rules
  rules:
  - alert: HostDown
    expr: up == 0
    for: 10m
    labels:
      severity: "critical"
    annotations:
      description: "监控的目标已丢失，请检查服务器自身或node-exporter服务"

  - alert: HostCpuUsage
    expr: 100-(avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) by(instance)* 100) > 90
    for: 10m
    labels:
      severity: "warning"
    annotations:
      summary: "{{ $labels.instance }} CPU使用率超过90%"
      description: "{{ $labels.instance }} CPU使用率超过90%,当前使用率:{{ printf \"%.2f\" $value }}%."


  - alert: HostMemoryUsage
    expr: 100 - ((node_memory_MemAvailable_bytes * 100) / node_memory_MemTotal_bytes) > 90
    for: 10m
    labels:
      severity: "warning"
    annotations:
      summary: "{{ $labels.instance }} 内存使用率超过90%"
      description: "{{ $labels.instance }} 内存使用率超过90%,当前使用率:{{ printf \"%.2f\" $value }}%."


  - alert: HostFileSystemUsage
    expr: (1-(node_filesystem_free_bytes{fstype=~"ext4|xfs",mountpoint!~".*tmp|.*boot" }/node_filesystem_size_bytes{fstype=~"ext4|xfs",mountpoint!~".*tmp|.*boot" }))*100 > 90
    for: 10m
    labels:
      severity: "warning"
    annotations:
      summary: "{{ $labels.instance }} 磁盘使用率超过90%"
      description: "{{ $labels.instance }} 磁盘使用率超过90%,当前使用率:{{ printf \"%.2f\" $value }}%."

  - alert: HighDiskIOUtilization
    expr: avg by (instance, device) (rate(node_disk_io_time_seconds_total[5m])) * 100 > 85
    for: 10m
    labels:
      severity: "warning"
    annotations:
      summary: "磁盘I/O利用率超过85% (instance {{ $labels.instance }}, device {{ $labels.device }})"
      description: "磁盘:{{ $labels.device }},主机:{{ $labels.instance }},I/O利用率超过85%。当前值:{{ printf \"%.2f\" $value }}%."
```

目录结构：

![image-2](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-2.png)

##### **AlertManager**

`systemd`服务文件：

```
[Unit]
Description=Alert Manager
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecStart=/opt/prometheus/alertmanager-0.25.0/alertmanager \
  --config.file=/opt/prometheus/alertmanager-0.25.0/alertmanager.yml \
  --storage.path=/opt/prometheus/alertmanager-0.25.0/data

Restart=always

[Install]
WantedBy=multi-user.target
```

配置文件：

```
global:
  resolve_timeout: 5m
  smtp_from: 'test@qq.com'
  smtp_smarthost: 'smtp.exmail.qq.com:465'
  smtp_auth_username: 'test@qq.com'
  smtp_auth_password: 'xxxxxxxxxx'
  smtp_require_tls: false

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 24h
  receiver: 'devops'

receivers:
- name: 'devops'
  webhook_configs:
  - url: 'http://192.168.0.47:8060/dingtalk/devops/send'
    send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```

##### prometheus-webhook-dingtalk

`systemd`服务文件：

```
[Unit]
Descripton=dingtalk
Documentation=https://github.com/timonwong/prometheus-webhook-dingtalk/
After=network.target
 
[Service]
Restart=on-failure
WorkingDirectory=/opt/prometheus/dingtalk-webhook-2.1.0
ExecStart=/opt/prometheus/dingtalk-webhook-2.1.0/prometheus-webhook-dingtalk --config.file=/opt/prometheus/dingtalk-webhook-2.1.0/config.yml --web.enable-ui
 
[Install]
WantedBy=multi-user.target
```

配置文件：

```
## Request timeout
# timeout: 5s

## Uncomment following line in order to write template from scratch (be careful!)
#no_builtin_template: true

## Customizable templates path
#templates:
#  - contrib/templates/legacy/template.tmpl
# 配置自定义钉钉消息模板
templates:
  #- /opt/prometheus/alertmanager-0.25.0/templates/dingtalk.tmpl
  - /opt/prometheus/dingtalk-webhook-2.1.0/templates/dingtalk.tmpl

## You can also override default template using `default_message`
## The following example to use the 'legacy' template from v0.3.0
#default_message:
#  title: '{{ template "legacy.title" . }}'
#  text: '{{ template "legacy.content" . }}'

## Targets, previously was known as "profiles"
# 告警分组 可以添加多个
targets:
  devops:
    url: https://oapi.dingtalk.com/robot/send?access_token=e8db2ec827bfac12ad83f0b5e52607a1ebadc66a8ca6f9be890fcc2fcab25c8a
    secret: SEC014af31f6f0cfcfa2cfc24724d74f4bc21fdd0a619955d19b7b676245e0c49b9
    #url: https://oapi.dingtalk.com/robot/send?access_token=f72c52b55350a51581d10f2c6eda378c2aa0fa8760d897f25e9f2607184c877d
    #secret: SEC1eca7e04aeffc5447ad36783a04780659476e0c125a4822308bfdaf5c7da7044
    message:
      title: '{{ template "dingtalk.title" . }}'
      text: '{{ template "dingtalk.content" . }}'
```

模板文件：

```
{{ define "__subject" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}]
{{ end }}


{{ define "__alert_list" }}{{ range . }}
---
**告警主机**： {{ .Labels.hostname }} ({{ .Labels.instance }})

**告警类型**： {{ .Labels.alertname }}

**告警级别**： {{ .Labels.severity }}

**告警详情**： {{ .Annotations.description }}

**告警时间**： {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
{{ end }}{{ end }}


{{ define "__resolved_list" }}{{ range . }}
---
**告警主机**： {{ .Labels.hostname }} ({{ .Labels.instance }})

**告警类型**： {{ .Labels.alertname }}

**告警级别**： {{ .Labels.severity }}

**告警详情**： {{ .Annotations.description }}

**告警时间**： {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}

**恢复时间**： {{ (.EndsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
{{ end }}{{ end }}


{{ define "dingtalk.title" }}
{{ template "__subject" . }}
{{ end }}


{{ define "dingtalk.content" }}
{{ if gt (len .Alerts.Firing) 0 }}
<font color=#FF0000> ======== **监控告警** ======== </font>
{{ template "__alert_list" .Alerts.Firing }}
---
{{ end }}


{{ if gt (len .Alerts.Resolved) 0 }}
<font color=#00800> ======== **恢复告警** ======== </font>
{{ template "__resolved_list" .Alerts.Resolved }}
---
{{ end }}
{{ end }}


{{ define "ding.link.title" }}{{ template "dingtalk.title" . }}{{ end }}
{{ define "ding.link.content" }}{{ template "dingtalk.content" . }}{{ end }}
{{ template "dingtalk.title" . }}
{{ template "dingtalk.content" . }}
```

##### **Grafana**

`systemd`服务文件：

```
[Unit]
Description=Grafana server
Documentation=http://docs.grafana.org
[Service]
Type=simple
User=prometheus
Group=prometheus
Restart=on-failure
ExecStart=/opt/prometheus/grafana-9.5.2/bin/grafana-server \
  --config=/opt/prometheus/grafana-9.5.2/conf/defaults.ini \
  --homepath=/opt/prometheus/grafana-9.5.2
[Install]
WantedBy=multi-user.target
```

##### **node-exporter**

`systemd`服务文件：

```
[Unit]
Description=node_exporter
Documentation=https://prometheus.io/
After=network.target
[Service]
ExecStart=/opt/prometheus/node_exporter-1.5.0/node_exporter
Restart=on-failure
[Install]
WantedBy=multi-user.target
```

### **3、基本使用**

#### **访问地址**

`Prometheus`：[http://192.168.0.47:9090](http://192.168.0.47:9090/)

`AlertManager`：[http://192.168.0.47:9093](http://192.168.0.47:9093/)

`Grafana`：[http://192.168.0.47:3000](http://192.168.0.47:3000/?orgId=1)

#### **Web页面展示**

##### **Prometheus**

![image-3](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-3.png)

![image-4](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-4.png)

##### **AlertManager**

![image-5](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-5.png)

##### **Grafana**

![image-6](https://raw.githubusercontent.com/zhime/picBed/main/images/2025/image-6.png)