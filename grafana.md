## Grafana+Loki+Prometheus
[Grafana官方文档](https://grafana.com/docs)
[Prometheus官方文档](https://prometheus.io/docs)

安装插件（用于收集docker日志）
````
docker plugin install grafana/loki-docker-driver:latest --alias loki --grant-all-permissions
````
升级插件
````
docker plugin disable loki --force
docker plugin upgrade loki grafana/loki-docker-driver:latest --grant-all-permissions
docker plugin enable loki
systemctl restart docker
````
卸载插件
````
docker plugin disable loki --force
docker plugin rm loki
````
###  docker-compose.yaml
````
version: "3.7"

x-logging:
  &loki-logging
  driver: loki
  options:ls
    loki-url: "http://localhost:3100/loki/api/v1/push"
    max-size: "50m"
    max-file: "10"
    loki-pipeline-stages: |
      - multiline:
          firstline: '^\[\d{2}:\d{2}:\d{2} \w{4}\]'

networks:
  loki:

services:
  loki:
    image: grafana/loki:2.3.0
    user: "root"
    restart: unless-stopped
    ports:
      - "3100:3100"
    volumes:
      - /root/loki:/loki
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - loki

#  promtail:
#    image: grafana/promtail:2.3.0
#    volumes:
#      - /var/log:/var/log
#    command: -config.file=/etc/promtail/config.yml
#    networks:
#      - loki

  grafana:
    image: grafana/grafana:latest
    user: "root"
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_SECURITY_ALLOW_EMBEDDING=true
    volumes:
      - /root/grafana/grafana:/var/lib/grafana
    # logging: *loki-logging 收集日志
    networks:
      - loki

  prometheus:
    image: prom/prometheus:latest
    user: "root"
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - /root/prometheus/config:/etc/prometheus
      - /root/prometheus/data:/prometheus

````
### prometheus.yml
````
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
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["10.23.50.140:9100"]

````
### node_exporter

1.软件包下载地址：https://prometheus.io/download

2.解压并将二进制文件拷贝至/usr/local/bin

3.新建node-exporter.service输入以下内容：
````
[Unit]
#服务描述
Description=Node Exporter Service
#指定了在systemd在执行完那些target之后再启动该服务
After=network.target

[Service]
#定义Service的运行类型
Type=simple

#定义systemctl start|stop|reload *.service 的执行方法（具体命令需要写绝对路径）
#注：ExecStartPre为启动前执行的命令
#ExecStartPre=/usr/bin/test "x${NETWORKMANAGER}" = xyes
ExecStart=/usr/local/bin/node_exporter --web.listen-address 0.0.0.0:9100
ExecReload=
ExecStop=

#创建私有的内存临时空间
PrivateTmp=True

[Install]
WantedBy=default.target
````

4.启动node_exporter:``systemctl start node-exporter.service

> 导入 Dashboard 模板

node_exporter 模板地址：https://grafana.com/dashboards/1860

### CPU计算参考
https://wiki.eryajf.net/pages/3814.html