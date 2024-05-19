# Monitoring and Alerting Nasa API

This document outlines the steps to set up a monitoring and alerting system for a web service using Prometheus, Grafana, and Alertmanager. The web service chosen for this example is the NASA API.

## Tasks

### 1. Choose a Web Service
For this example, we are monitoring the NASA API available at [https://api.nasa.gov/](https://api.nasa.gov/).

### 2. Set Up Monitoring Tools

#### Prometheus
Prometheus is used to scrape and store metrics from the NASA API.

#### Grafana
Grafana is used to visualize the metrics collected by Prometheus.

#### Alertmanager
Alertmanager is used to handle alerts sent by Prometheus.

### Steps to Set Up

1. **Install Prometheus, Grafana, and Alertmanager:**
    - Install Prometheus from [Prometheus.io](https://prometheus.io/download/)
    - Install node_exporter from [Prometheus.io](https://prometheus.io/download/)
    - Install blackbox_exporter from [Prometheus.io](https://prometheus.io/download/)
    - Install Grafana from [Grafana.com](https://grafana.com/grafana/download)
    - Install Alertmanager from [Prometheus.io](https://prometheus.io/download/#alertmanager)

### Install Prometheus
Use the following commands to install Prometheus as a service
```bash
cd /tmp
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.45.5/prometheus-2.45.5.linux-amd64.tar.gz
tar -xvf prometheus-2.45.5.linux-amd64.tar.gz
cd prometheus-2.45.5.linux-amd64
sudo mv prometheus /usr/local/bin/
sudo mv promtool /usr/local/bin/
sudo mkdir /etc/prometheus
sudo mv prometheus.yml /etc/prometheus/
sudo mv consoles /etc/prometheus/
sudo mv console_libraries /etc/prometheus/
sudo useradd --no-create-home --shell /bin/false prometheus
sudo chown -R prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```

```bash
sudo tee /etc/systemd/system/prometheus.service <<EOF
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file /etc/prometheus/prometheus.yml \
  --storage.tsdb.path /var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
EOF
```


```bash
chmod -R 777 /etc/prometheus
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
sudo systemctl status prometheus
```

### Install Grafana
```bash
sudo apt-get install -y adduser libfontconfig1 musl
wget https://dl.grafana.com/enterprise/release/grafana-enterprise_11.0.0_amd64.deb
sudo dpkg -i grafana-enterprise_11.0.0_amd64.deb
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
sudo systemctl status grafana-server
```
### Install Node Exporter

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz
tar xvfz node_exporter-1.8.0.linux-amd64.tar.gz
sudo mv node_exporter-1.8.0.linux-amd64/node_exporter /usr/local/bin/
```
```bash
sudo tee /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```
### Install Blackbox Exporter

```bash

wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz
tar xvfz blackbox_exporter-0.25.0.linux-amd64.tar.gz
sudo mv blackbox_exporter-0.25.0.linux-amd64/blackbox_exporter /usr/local/bin/
sudo mkdir -p /etc/blackbox_exporter
```

```bash
sudo tee /etc/systemd/system/alertmanager.service > /dev/null <<EOF
[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/alertmanager --config.file=/etc/alertmanager/alertmanager.yml

[Install]
WantedBy=multi-user.target
EOF
```

```bash

sudo tee /etc/blackbox_exporter/blackbox.yml > /dev/null <<EOF
modules:
  nasa_api:
    prober: http
    timeout: 10s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2"]
      valid_status_codes: [200]
      preferred_ip_protocol: "ip4"
      method: GET
      tls_config:
        insecure_skip_verify: false
      headers:
        Accept: application/json
```
```bash

sudo systemctl daemon-reload
sudo systemctl enable blackbox_exporter
sudo systemctl start blackbox_exporter
sudo systemctl status blackbox_exporter
```
### Install Alertmanager
```bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
tar xvfz alertmanager-0.27.0.linux-amd64.tar.gz
sudo mv alertmanager-0.27.0.linux-amd64/alertmanager /usr/local/bin/
sudo mkdir -p /etc/alertmanager
```

```bash
sudo tee /etc/alertmanager/alertmanager.yml > /dev/null <<EOF
[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/alertmanager --config.file=/etc/alertmanager/alertmanager.yml

[Install]
WantedBy=multi-user.target
EOF
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
sudo systemctl status alertmanager
```

2. **Configure Prometheus:**
    - Edit the `prometheus.yml` configuration file to include the following scrape configurations:

```yaml
global:

alerting:
  alertmanagers:
    - static_configs:
        - targets:
           - 127.0.0.1:9093

rule_files:
  - "alert-rules.yml"

scrape_configs:
  - job_name: "prometheus"


    static_configs:
      - targets: ["localhost:9090"]


  - job_name: 'blackbox_Nasa_API'
    metrics_path: /probe
    params:
    static_configs:
      - targets:
          - https://api.nasa.gov/planetary/apod?api_key=LqkYnxsKYJgcF0Hy1Defx1lULvoTe7Mcrsx1K7DX

    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
      - source_labels: [instance]
        target_label: instance_short
        replacement: "api.nasa.gov"


  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```

3. **Configure Grafana:**
    - Add Prometheus as a data source in Grafana.
    - Create dashboards to visualize the metrics from Prometheus.

4. **Configure Alertmanager:**
    - Edit the `alertmanager.yml` configuration file to include email notifications:

```yaml
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: 'email-notifications'



receivers:
  - name: 'email-notifications'
    email_configs:
    - to: 'imkobi5@gmail.com'  # Replace with your recipient email address
      from: 'imkobi5@gmail.com'  # Replace with your SendGrid verified sender email address
      smarthost: smtp.sendgrid.net:587
      auth_username: 'apikey'  # Replace with your SendGrid API key
      auth_password: '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'  # Replace with your SendGrid API key
      require_tls: true



inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']    
```

5. **Define Alerting Rules:**
    - Create an `alert-rules.yml` file with the following content:

 ```yaml
groups:
  - name: Nasa_api_alerts & node_alerts
    rules:
      - alert: HighResponseTime
        expr: probe_duration_seconds{instance="https://api.nasa.gov/planetary/apod?api_key=LqkYnxsKYJgcF0Hy1Defx1lULvoTe7Mcrsx1K7DX", instance_short="api.nasa.gov", job="blackbox_Nasa_API"} > 7
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "High response time detected"
          description: "The response time for {{ $labels.instance }} is above 0.5 seconds."

      - alert: APIErrorRate
        expr: probe_http_status_code != 200
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "API error rate detected"
          description: "The API at {{ $labels.instance }} is returning non-200 status codes."

  - name: node_exporter_alerts
    rules:
      - alert: HighCpuUsage
        expr: 100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "The CPU usage on {{ $labels.instance }} is above 80%."

      - alert: HighMemoryUsage
        expr: node_memory_Active_bytes / node_memory_MemTotal_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
          description: "The memory usage on {{ $labels.instance }} is above 90%."
```

### 3. Simulate Incidents
To triger Alertmanager, add the following line to /etc/hosts:
```bash
109.94.209.70     api.nasa.gov
```
It will triger Alermanger and an e-mail will be sent:
![App Screenshot](https://i.postimg.cc/VYRkWwdy/etoro.png)

## Documentation

### Setup and Configuration

- **Prometheus**: Collects metrics from the NASA API and Node Exporter.
- **Grafana**: Visualizes the metrics collected by Prometheus.
- **Alertmanager**: Handles alerts sent by Prometheus.

### Metrics Monitored

- HTTP response time for the NASA API.
- Error rate for the NASA API.
- CPU and Memory usage of the server.

![App Screenshot](https://i.postimg.cc/LsCRKH1n/grafana.png)


### Alerting Rules

- **HighResponseTime**: Triggers if the response time for the NASA API is above 0.5 seconds.
- **APIErrorRate**: Triggers if the NASA API returns non-200 status codes.
- **HighCpuUsage**: Triggers if the CPU usage is above 80%.
- **HighMemoryUsage**: Triggers if the memory usage is above 90%.

### Accessing the Monitoring Dashboard and Viewing Alerts

1. **Grafana Dashboard**: Access Grafana at `http://localhost:3000` and view the dashboards for the NASA API and server metrics.
2. **Alertmanager**: Access Alertmanager at `http://localhost:9093` to view and manage alerts.

### Additional Instructions

- Replace `YOUR_API_KEY` in the Prometheus configuration with your actual NASA API key.
- Ensure that email notifications are configured with your email credentials in Alertmanager.
- Adjust the monitoring and alerting rules as needed for your specific use case.
