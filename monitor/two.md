# Linux Server Monitoring Using Prometheus, Node Exporter and Grafana on Ubuntu WSL

## 1. Project Overview

### Project Objective

In this project, we will implement a Linux server monitoring solution using three open-source monitoring tools:

1. **Node Exporter** – Collects Linux system metrics such as CPU, memory, disk and network statistics.
2. **Prometheus** – Scrapes, collects and stores metrics exposed by Node Exporter.
3. **Grafana** – Connects to Prometheus and visualizes the collected metrics through dashboards.

We are implementing this monitoring solution directly on Ubuntu running inside WSL2.

We are not using Docker for this monitoring setup.

### Project Environment

| Component | Version / Configuration |
|---|---|
| Operating System | Ubuntu 26.04.1 LTS (WSL2) |
| Host Machine | Windows 11 |
| Node Exporter | 1.10.2 |
| Prometheus | 3.16.0 |
| Grafana | Ubuntu APT installation planned |
| Node Exporter Port | 9100 |
| Prometheus Port | 9090 |
| Grafana Port | 3000 |
| Deployment Method | Direct Linux installation |
| Container Usage | Not used for monitoring |

## 2. Monitoring Architecture

### High-Level Architecture

```text
                 WINDOWS 11
                      |
                      |
              Ubuntu WSL2
                      |
          +-----------+-----------+
          |                       |
          |                       |
    Node Exporter             Prometheus
       :9100                    :9090
          |                       |
          |                       |
    Reads Linux             Scrapes Metrics
    System Statistics       Every 15 Seconds
          |                       |
          +------ HTTP GET <-----+
                                  |
                                  |
                           Stores Metrics
                           In TSDB Database
                                  |
                                  |
                              Grafana
                               :3000
                                  |
                                  |
                         Queries Prometheus
                            HTTP API
                                  |
                                  |
                          Monitoring Dashboard
                                  |
                                  |
                       CPU / Memory / Disk
```

### Component Responsibilities

| Component | Responsibility |
|---|---|
| Node Exporter | Exposes Linux system metrics |
| Prometheus | Collects and stores time-series metrics |
| Grafana | Queries Prometheus and displays dashboards |

**Important:** Node Exporter does not push metrics to Prometheus. Prometheus follows a pull-based monitoring architecture.

Grafana also does not directly collect metrics from Node Exporter. It queries the Prometheus data source.

---

## 3. How Node Exporter Collects Linux Metrics

This is one of the most important concepts to understand.

Node Exporter is an exporter developed for Prometheus monitoring.

It runs as a process on the Linux machine and collects operating system statistics.

### Where does Node Exporter get the data?

Linux exposes system statistics through special virtual filesystems.

Two important locations are:

**1. /proc filesystem**

Contains kernel and process-related information.

Examples:

| File | Information |
|---|---|
| /proc/stat | CPU statistics |
| /proc/meminfo | Memory statistics |
| /proc/loadavg | System load average |
| /proc/net/dev | Network statistics |
| /proc/vmstat | Virtual memory statistics |

**2. /sys filesystem**

Provides information about devices and kernel subsystems.

Examples:

| Directory | Information |
|---|---|
| /sys/class/net | Network interfaces |
| /sys/class/block | Block devices |
| /sys/class/thermal | Thermal information |
| /sys/class/power_supply | Power supply information |

Node Exporter uses its collectors to read supported system statistics from these interfaces and exposes them as Prometheus metrics.

### Example: CPU collection

Linux maintains CPU counters in:

```bash
cat /proc/stat
```

Node Exporter's CPU collector reads these statistics and exposes metrics such as:

```text
node_cpu_seconds_total
```

### Example: Memory collection

Linux maintains memory information in:

```bash
cat /proc/meminfo
```

Node Exporter exposes metrics such as:

```text
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes
node_memory_MemFree_bytes
```

### How can we see the exported metrics?

Node Exporter exposes an HTTP endpoint:

```text
http://localhost:9100/metrics
```

Test it:

```bash
curl http://localhost:9100/metrics
```

Filter CPU metrics:

```bash
curl -s http://localhost:9100/metrics | grep node_cpu
```

Filter memory metrics:

```bash
curl -s http://localhost:9100/metrics | grep node_memory
```

**Remember:**

Node Exporter collects system statistics and exposes them through an HTTP endpoint. It does not maintain a time-series database.

---

## 4. Node Exporter Installation

### Step 1: Create monitoring directory

```bash
mkdir -p ~/monitoring
cd ~/monitoring
```

### Step 2: Download Node Exporter

Download the Linux AMD64 release from the official Prometheus download page.

Extract the downloaded archive:

```bash
tar -xvf node_exporter-1.10.2.linux-amd64.tar.gz
```

Navigate into the extracted directory:

```bash
cd node_exporter-1.10.2.linux-amd64
```

### Step 3: Verify executable

```bash
ls -l
```

Expected:

```text
LICENSE
NOTICE
node_exporter
```

### Step 4: Start Node Exporter

```bash
./node_exporter &
```

### Step 5: Verify process

```bash
ps -ef | grep node_exporter
```

### Step 6: Verify listening port

```bash
ss -tulnp | grep 9100
```

Expected:

```text
tcp LISTEN 0 4096 *:9100 *:* users:(("node_exporter",...))
```

### Step 7: Verify metrics endpoint

```bash
curl http://localhost:9100/metrics
```

Node Exporter should now expose Linux metrics.

### Important Note

If you receive:

```text
listen tcp :9100: bind: address already in use
```

It means another process is already listening on port 9100.

Check the existing process:

```bash
pgrep -af node_exporter
```

Do not start another instance unnecessarily.

---

## 5. How Prometheus Collects Metrics

Now let's understand how Prometheus communicates with Node Exporter.

Prometheus follows a pull-based architecture.

The communication happens as follows:

1. Node Exporter reads Linux system statistics.
2. Node Exporter exposes those statistics through port 9100.
3. Prometheus reads its configuration file.
4. Prometheus identifies Node Exporter as a monitoring target.
5. Prometheus sends an HTTP GET request to Node Exporter.
6. Node Exporter responds with metrics in Prometheus exposition format.
7. Prometheus parses the metrics.
8. Prometheus stores the collected metrics in its local time-series database (TSDB).

### Example HTTP Communication

Prometheus sends:

```http
GET /metrics HTTP/1.1
Host: localhost:9100
```

Node Exporter responds with:

```text
node_cpu_seconds_total{cpu="0",mode="idle"} 123456
node_memory_MemAvailable_bytes 2048000000
node_memory_MemTotal_bytes 4096000000
```

Prometheus stores these metrics with timestamps and labels.

### Important Concept

Node Exporter is the metrics endpoint.

Prometheus is the metrics collector and storage engine.

Grafana is the visualization layer.

---

## 6. Prometheus Installation

### Step 1: Download Prometheus

Download the Linux AMD64 release from the official Prometheus downloads page.

Extract:

```bash
tar -xvf prometheus-3.16.0.linux-amd64.tar.gz
```

Navigate:

```bash
cd prometheus-3.16.0.linux-amd64
```

### Step 2: Verify files

```bash
ls -l
```

Expected:

```text
LICENSE
NOTICE
prometheus
prometheus.yml
promtool
```

### Step 3: Understand important files

| File | Purpose |
|---|---|
| prometheus | Prometheus server executable |
| prometheus.yml | Monitoring configuration |
| promtool | Configuration validation utility |
| data/ | Local time-series database storage |
| LICENSE | License information |
| NOTICE | Third-party notices |

### Step 4: Configure prometheus.yml

Open the configuration file:

```bash
nano prometheus.yml
```

Use the following configuration:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files: []

scrape_configs:

  # Prometheus monitoring itself
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          app: "prometheus"

  # Linux server monitoring
  - job_name: "node-exporter"
    static_configs:
      - targets: ["localhost:9100"]
        labels:
          app: "node-exporter"
```

### Understanding the Configuration

**global**

Defines common settings for Prometheus.

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
```

- scrape_interval: Prometheus collects metrics every 15 seconds.
- evaluation_interval: Prometheus evaluates recording and alerting rules every 15 seconds.

**scrape_configs**

Defines which monitoring targets Prometheus should collect metrics from.

**job_name**

```yaml
- job_name: "node-exporter"
```

Defines the monitoring job name.

**static_configs**

```yaml
static_configs:
  - targets: ["localhost:9100"]
```

Defines the target address that Prometheus should scrape.

Since both Prometheus and Node Exporter are running directly on the same WSL machine, localhost is used.

### Step 5: Validate Prometheus configuration

```bash
./promtool check config prometheus.yml
```

Expected:

```text
Checking prometheus.yml
SUCCESS: prometheus.yml is valid prometheus config file syntax
```

### Step 6: Start Prometheus

```bash
./prometheus \
  --config.file=prometheus.yml \
  --storage.tsdb.path=./data &
```

### Step 7: Verify Prometheus

```bash
curl http://localhost:9090/-/ready
```

Expected:

```text
Prometheus Server is Ready.
```

Check Prometheus metrics:

```bash
curl http://localhost:9090/metrics
```

Open the Prometheus UI:

```text
http://localhost:9090
```

---

## 7. Understanding Prometheus Target Health

Prometheus provides a Targets page to check the health of monitoring endpoints.

Open:

```text
http://localhost:9090/targets
```

Expected monitoring targets:

| Job | Endpoint | Expected Status |
|---|---|---|
| prometheus | localhost:9090 | UP |
| node-exporter | localhost:9100 | UP |

### What does UP mean?

UP means Prometheus successfully scraped the target.

### What does DOWN mean?

DOWN means Prometheus could not successfully scrape the target.

Possible reasons:

- Exporter process is stopped.
- Incorrect target IP or port.
- Network connectivity problem.
- Firewall restrictions.
- Incorrect metrics endpoint.

### Troubleshooting commands

```bash
ss -tulnp | grep 9100
```

```bash
curl http://localhost:9100/metrics
```

```bash
pgrep -af node_exporter
```

Validate configuration:

```bash
./promtool check config prometheus.yml
```

---

## 8. Understanding Prometheus Time-Series Database (TSDB)

Prometheus stores collected metrics in its local time-series database.

A time series consists of:

- Metric name
- Labels
- Timestamp
- Value

Example:

```text
node_memory_MemAvailable_bytes{
  instance="localhost:9100",
  job="node-exporter"
} 2048000000
```

Here:

| Field | Value |
|---|---|
| Metric | node_memory_MemAvailable_bytes |
| Instance | localhost:9100 |
| Job | node-exporter |
| Value | 2048000000 |
| Unit | Bytes |

Every scrape produces new samples.

Prometheus stores these samples over time so we can query historical metrics and visualize trends.

Our local storage directory:

```text
prometheus-3.16.0.linux-amd64/data/
```

Check its contents:

```bash
ls -lh data/
```

**Important:** Do not delete this directory unless you intentionally want to remove the locally stored Prometheus monitoring data.

---

## 9. PromQL – Monitoring CPU and Memory

PromQL is the query language used to retrieve and calculate metrics stored in Prometheus.

### CPU Metrics

Raw CPU metric:

```promql
node_cpu_seconds_total
```

CPU usage percentage:

```promql
100 * (1 - avg by(instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
))
```

Explanation:

- node_cpu_seconds_total: CPU time counters.
- mode="idle": CPU time spent idle.
- rate(): Calculates the per-second counter increase.
- avg by(instance): Calculates the average across CPU cores.
- 1 - idle: Calculates non-idle CPU usage.
- Multiply by 100: Converts the result into a percentage.

### Memory Metrics

Total memory:

```promql
node_memory_MemTotal_bytes
```

Available memory:

```promql
node_memory_MemAvailable_bytes
```

Used memory:

```promql
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

Memory utilization percentage:

```promql
100 * (
  1 -
  node_memory_MemAvailable_bytes
  /
  node_memory_MemTotal_bytes
)
```

### Additional useful metrics

CPU cores:

```promql
count(count by(cpu) (node_cpu_seconds_total))
```

System load:

```promql
node_load1
```

Filesystem available:

```promql
node_filesystem_avail_bytes
```

Network received traffic:

```promql
rate(node_network_receive_bytes_total[5m])
```

Network transmitted traffic:

```promql
rate(node_network_transmit_bytes_total[5m])
```

---

## 10. How Grafana Gets Data From Prometheus

This is another important concept.

Grafana does not directly read Linux files or collect CPU and memory metrics from Node Exporter.

Grafana connects to Prometheus as a data source.

### Communication Flow

```text
Linux Kernel
     |
     | System statistics
     v
Node Exporter
     |
     | HTTP GET /metrics
     | Prometheus scrapes
     v
Prometheus
     |
     | Stores time-series data
     v
Prometheus TSDB
     |
     | Grafana sends PromQL queries
     v
Grafana
     |
     | Receives query results
     v
CPU / Memory Dashboards
```

### Step-by-Step Data Flow

**Step 1: Node Exporter exposes metrics**

```text
http://localhost:9100/metrics
```

**Step 2: Prometheus scrapes metrics**

```text
http://localhost:9090
```

Prometheus stores the collected metrics.

**Step 3: Grafana connects to Prometheus**

Grafana uses Prometheus as its data source.

**Step 4: Grafana sends queries**

For example:

```promql
100 * (1 - avg by(instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
))
```

**Step 5: Prometheus returns query results**

Grafana receives the results through the Prometheus HTTP API.

**Step 6: Grafana visualizes the results**

Grafana displays the results using:

- Time-series graphs
- Gauge panels
- Stat panels
- Tables
- Bar charts

**Remember:**

Prometheus collects and stores metrics.

Grafana queries and visualizes those metrics.

Grafana does not replace Prometheus.

---

## 11. Grafana Installation on Ubuntu WSL

We are using the official Grafana APT repository.

### Step 1: Install prerequisites

```bash
sudo apt-get install -y apt-transport-https wget gnupg
```

### Step 2: Add Grafana GPG key

```bash
sudo mkdir -p /etc/apt/keyrings
```

```bash
sudo wget -O /etc/apt/keyrings/grafana.asc \
https://apt.grafana.com/gpg-full.key
```

```bash
sudo chmod 644 /etc/apt/keyrings/grafana.asc
```

### Step 3: Add Grafana repository

```bash
echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" \
| sudo tee /etc/apt/sources.list.d/grafana.list
```

### Step 4: Update package information

```bash
sudo apt-get update
```

### Step 5: Install Grafana

```bash
sudo apt-get install -y grafana
```

### Step 6: Start Grafana

Check whether systemd is running:

```bash
systemctl is-system-running
```

If systemd is active:

```bash
sudo systemctl enable --now grafana-server
```

Check service:

```bash
sudo systemctl status grafana-server
```

If systemd is not active, configure systemd support in WSL or use the service startup method supported by your WSL installation.

### Step 7: Access Grafana

Open:

```text
http://localhost:3000
```

Default login:

```text
Username: admin
Password: admin
```

Grafana will ask you to change the default password after the first login.

---

## 12. Configure Prometheus as Grafana Data Source

After logging into Grafana:

1. Open Connections.
2. Select Data Sources.
3. Click Add data source.
4. Select Prometheus.
5. Enter the following URL:

```text
http://localhost:9090
```

6. Click Save & Test.

Expected:

```text
Successfully queried the Prometheus API.
```

Now Grafana can query metrics stored by Prometheus.

---

## 13. Create Linux CPU and Memory Dashboard

We will create a simple Linux monitoring dashboard.

### Panel 1: CPU Utilization

Query:

```promql
100 * (1 - avg by(instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
))
```

Visualization:

```text
Gauge / Time Series
```

Unit:

```text
Percent (0-100)
```

### Panel 2: Memory Utilization

Query:

```promql
100 * (
  1 -
  node_memory_MemAvailable_bytes
  /
  node_memory_MemTotal_bytes
)
```

Visualization:

```text
Gauge / Time Series
```

Unit:

```text
Percent (0-100)
```

### Panel 3: Total Memory

```promql
node_memory_MemTotal_bytes
```

Unit:

```text
Bytes (IEC)
```

### Panel 4: Available Memory

```promql
node_memory_MemAvailable_bytes
```

Unit:

```text
Bytes (IEC)
```

### Panel 5: CPU Usage Over Time

```promql
100 * (1 - avg by(instance) (
  rate(node_cpu_seconds_total{mode="idle"}[1m])
))
```

Visualization:

```text
Time Series
```

### Dashboard Layout

```text
+-----------------------------------------------+
|          LINUX SERVER MONITORING              |
+----------------------+------------------------+
|    CPU UTILIZATION   |   MEMORY UTILIZATION    |
|                      |                        |
|         35%          |          62%           |
+----------------------+------------------------+
|             CPU USAGE OVER TIME               |
|                                               |
|             Time Series Graph                 |
+-----------------------------------------------+
|              MEMORY INFORMATION               |
|                                               |
|       Total Memory | Available Memory         |
+-----------------------------------------------+
```

---

## 14. Troubleshooting and Real-Time Scenarios

### Scenario 1: Node Exporter is not running

Check:

```bash
ps -ef | grep node_exporter
```

Check port:

```bash
ss -tulnp | grep 9100
```

Restart if required:

```bash
cd ~/monitoring/node_exporter-1.10.2.linux-amd64
./node_exporter &
```

### Scenario 2: Prometheus target is DOWN

Check Node Exporter:

```bash
curl http://localhost:9100/metrics
```

Check Prometheus configuration:

```bash
./promtool check config prometheus.yml
```

Check target health:

```text
http://localhost:9090/targets
```

### Scenario 3: Prometheus is not starting

Check whether port 9090 is occupied:

```bash
sudo ss -tulnp | grep 9090
```

Check running processes:

```bash
pgrep -af prometheus
```

Do not start another Prometheus instance if one is already using port 9090.

### Scenario 4: Grafana cannot connect to Prometheus

Check Prometheus:

```bash
curl http://localhost:9090/-/ready
```

Check Grafana:

```bash
sudo systemctl status grafana-server
```

Check port:

```bash
ss -tulnp | grep 3000
```

Verify the Grafana data source URL:

```text
http://localhost:9090
```

### Scenario 5: No CPU or memory metrics

Check Node Exporter:

```bash
curl -s http://localhost:9100/metrics | grep node_memory
```

Check Prometheus:

```promql
up{job="node-exporter"}
```

Expected:

```text
1
```

If the result is 0, check the exporter and scrape target.

---

## 15. Important Production Concepts

### Does Node Exporter push metrics to Prometheus?

No. Node Exporter exposes metrics through an HTTP endpoint. Prometheus periodically pulls metrics from that endpoint.

### Does Grafana collect metrics directly from Node Exporter?

No. Grafana queries Prometheus, which has already collected and stored the metrics.

### What happens if Node Exporter stops?

Prometheus cannot scrape the target. The target becomes DOWN, and the `up` metric for that target becomes 0.

### What happens if Prometheus stops?

New metrics are not collected during the outage. Existing locally stored metrics remain available after restart, provided the storage directory is preserved.

### What happens if Grafana stops?

Metrics collection by Prometheus continues. Grafana dashboards become unavailable until Grafana starts again.

### Does Prometheus store metrics permanently?

No. Prometheus uses retention settings to manage stored data. Data is retained according to its configured retention policy and available storage.

### Can we monitor multiple Linux servers?

Yes. Install Node Exporter on each Linux server and configure the Prometheus scrape targets with their reachable IP addresses and port 9100.

Example:

```yaml
- job_name: "linux-servers"
  static_configs:
    - targets:
        - "10.0.1.10:9100"
        - "10.0.1.11:9100"
        - "10.0.1.12:9100"
```

In production, allow access to Node Exporter only from trusted monitoring systems.

---

## 16. Future Enhancements

After completing the basic CPU and memory monitoring, we can extend this setup.

| Tool | Purpose |
|---|---|
| Blackbox Exporter | Application and endpoint availability |
| Alertmanager | Alert routing and notifications |
| Grafana | Advanced dashboards |
| Node Exporter | Linux server metrics |
| Prometheus | Metrics collection and storage |

### Future Architecture

```text
                 Linux Servers
                       |
                Node Exporter
                       |
                       v
                 Prometheus
                       |
              +--------+--------+
              |                 |
              v                 v
           Grafana          Alertmanager
              |
              v
          Dashboards


             Applications
                  |
                  v
           Blackbox Exporter
                  |
                  v
              Prometheus
```

---

## 17. Project Validation Checklist

- [x] Ubuntu WSL environment prepared.
- [x] Node Exporter 1.10.2 extracted.
- [x] Node Exporter running on port 9100.
- [x] Node Exporter metrics endpoint verified.
- [x] Prometheus 3.16.0 extracted.
- [x] Prometheus configuration created.
- [x] Prometheus configuration validated.
- [x] Prometheus running on port 9090.
- [x] Prometheus target health verified.
- [x] Node Exporter target showing UP.
- [x] Prometheus target showing UP.
- [ ] Grafana installed and configured.
- [ ] Prometheus added as Grafana data source.
- [ ] CPU dashboard created.
- [ ] Memory dashboard created.
- [ ] Blackbox Exporter configured.

---

## 18. Important Commands – Quick Reference

| Requirement | Command |
|---|---|
| Check Node Exporter process | `pgrep -af node_exporter` |
| Check Prometheus process | `pgrep -af prometheus` |
| Check port 9100 | `ss -tulnp \| grep 9100` |
| Check port 9090 | `ss -tulnp \| grep 9090` |
| View Node Exporter metrics | `curl localhost:9100/metrics` |
| Check Prometheus readiness | `curl localhost:9090/-/ready` |
| Validate Prometheus YAML | `./promtool check config prometheus.yml` |
| Check Prometheus targets | `http://localhost:9090/targets` |
| Check Grafana service | `sudo systemctl status grafana-server` |
| Open Grafana | `http://localhost:3000` |

---

## 19. Learning Outcomes

After completing this project, we will understand:

1. How Linux exposes CPU and memory statistics.
2. How Node Exporter collects system metrics.
3. How Prometheus scrapes metrics using HTTP.
4. How Prometheus stores time-series data.
5. How PromQL calculates CPU and memory utilization.
6. How Grafana connects to Prometheus.
7. How Grafana queries Prometheus and creates dashboards.
8. How to troubleshoot monitoring targets.
9. How to extend monitoring to multiple Linux servers.
10. How Blackbox Exporter can be used for application availability monitoring.

## 20. Official Documentation

- [Prometheus Documentation](https://prometheus.io/docs/introduction/first_steps/)
- [Prometheus Node Exporter](https://github.com/prometheus/node_exporter)
- [Node Exporter Linux Monitoring Guide](https://prometheus.io/docs/guides/node-exporter/)
- [Prometheus Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Grafana Installation on Ubuntu](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/)
- [Grafana Prometheus Data Source](https://grafana.com/docs/grafana/latest/datasources/prometheus/configure/)

---

**Project Status:** Prometheus and Node Exporter successfully configured and verified on Ubuntu WSL2. Grafana installation and dashboard configuration are the next implementation steps.

**Learning Approach:** Theory → Installation → Configuration → Metric Collection → PromQL → Dashboard → Troubleshooting → Production Scenarios.
