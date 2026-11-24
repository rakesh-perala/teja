# Linux Server Monitoring Using Prometheus, Node Exporter and Grafana

## 1. Project Overview

In this project, we will monitor a local Ubuntu Linux system using three popular open-source monitoring tools:

1. **Node Exporter** – Collects Linux system metrics.
2. **Prometheus** – Collects and stores metrics.
3. **Grafana** – Visualizes metrics using dashboards.

### Business Requirement

Imagine a company has multiple Linux servers running production applications.

The DevOps team needs to monitor:

- CPU utilization
- Memory utilization
- Disk usage
- Filesystem information
- Network traffic
- System performance

**Question:** How can a DevOps Engineer monitor Linux servers and identify performance issues?

**Solution:** Use Node Exporter, Prometheus and Grafana.

---

## 2. Monitoring Architecture

```text
                 UBUNTU LINUX SERVER
                (Local WSL Environment)
                         |
                         |
                         v
                 +---------------+
                 | Node Exporter |
                 |   Port 9100   |
                 +---------------+
                         |
                         |
                  Exposes Metrics
                         |
                         v
                 +---------------+
                 |  Prometheus   |
                 |   Port 9090   |
                 +---------------+
                         |
                         |
                   Stores Metrics
                         |
                         v
                 +---------------+
                 |    Grafana    |
                 |   Port 3000   |
                 +---------------+
                         |
                         v
                  Linux Dashboard
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
         CPU           Memory          Disk
       Metrics         Metrics        Metrics
```

### How does this architecture work?

**Step 1 – Node Exporter**

Node Exporter runs on the Linux machine and collects system-level metrics.

Example:
- CPU usage
- Available memory
- Disk space
- Network statistics

It exposes metrics through port `9100`.

**Step 2 – Prometheus**

Prometheus connects to Node Exporter and periodically collects the available metrics.

This process is called **scraping**.

Prometheus stores the collected metrics in its time-series database.

**Step 3 – Grafana**

Grafana connects to Prometheus as a data source.

Grafana uses Prometheus metrics to display dashboards and graphs.

---

## 3. Environment Details

| Component | Version / Configuration |
|---|---|
| Operating System | Ubuntu WSL |
| Node Exporter | 1.10.2 |
| Prometheus | 3.16.0 |
| Grafana | Installed and running |
| Node Exporter Port | 9100 |
| Prometheus Port | 9090 |
| Grafana Port | 3000 |
| Dashboard ID | 1860 |

### Project Directory Structure

```text
~/monitoring/
|
└── node_exporter-1.10.2.linux-amd64/
    |
    ├── node_exporter
    |
    └── prometheus-3.16.0.linux-amd64/
        |
        ├── prometheus
        ├── promtool
        ├── prometheus.yml
        ├── data/
        ├── LICENSE
        └── NOTICE
```

---

# PART 1 – NODE EXPORTER INSTALLATION

## 4. What is Node Exporter?

Node Exporter is a Prometheus exporter used to collect hardware and operating system metrics from Linux systems.

It collects information such as:

- CPU statistics
- Memory information
- Disk usage
- Filesystem statistics
- Network information
- System load

### Step 1: Download Node Exporter

Download the Linux AMD64 release from the official Prometheus download page.

https://prometheus.io/download/

Extract the downloaded archive.

Example:

```bash
cd ~/monitoring
```

```bash
tar -xvf node_exporter-1.10.2.linux-amd64.tar.gz
```

### Step 2: Navigate to Node Exporter Directory

```bash
cd ~/monitoring/node_exporter-1.10.2.linux-amd64
```

Check the files:

```bash
ls -l
```

You should see the `node_exporter` executable.

### Step 3: Start Node Exporter

In our WSL environment, we excluded problematic filesystem mount points to prevent duplicate filesystem metrics.

Execute:

```bash
./node_exporter --collector.filesystem.mount-points-exclude='^/(dev|proc|run/user($|/)|run/credentials/.+|sys|var/lib/docker/.+)($|/)'
```

**Explanation:**

- `./node_exporter` – Starts Node Exporter.
- `--collector.filesystem.mount-points-exclude` – Excludes selected filesystem mount points.
- `/run/user` – Excluded because our WSL environment reported duplicate filesystem metrics for this mount point.

Keep this terminal open while Node Exporter is running.

### Step 4: Verify Node Exporter

Check the listening port:

```bash
sudo ss -lntp | grep 9100
```

Expected result:

```text
LISTEN 0 4096 *:9100 *:*
```

Check the metrics endpoint:

```bash
curl http://localhost:9100/metrics
```

You should see metrics similar to:

```text
node_cpu_seconds_total
node_memory_MemAvailable_bytes
node_memory_MemTotal_bytes
node_filesystem_size_bytes
node_network_receive_bytes_total
```

Check the HTTP status:

```bash
curl -sS -o /dev/null -w "%{http_code}\n" http://localhost:9100/metrics
```

Expected output:

```text
200
```

**What does HTTP 200 mean?**

It means the Node Exporter metrics endpoint is responding successfully.

---

# PART 2 – PROMETHEUS INSTALLATION

## 5. What is Prometheus?

Prometheus is an open-source monitoring and alerting system.

It collects metrics from different targets and stores them as time-series data.

### Important Prometheus Features

- Metrics collection
- Time-series database
- PromQL query language
- Service monitoring
- Alerting integration
- Target health monitoring

### Step 1: Download Prometheus

Download the Linux AMD64 release from:

https://prometheus.io/download/

Extract the archive.

```bash
cd ~/monitoring
```

```bash
tar -xvf prometheus-3.16.0.linux-amd64.tar.gz
```

### Step 2: Navigate to Prometheus Directory

```bash
cd ~/monitoring/node_exporter-1.10.2.linux-amd64/prometheus-3.16.0.linux-amd64
```

Check the files:

```bash
ls -l
```

Important files:

| File | Purpose |
|---|---|
| prometheus | Prometheus executable |
| prometheus.yml | Main configuration file |
| promtool | Configuration and rule validation tool |
| data/ | Local TSDB data directory |

### Step 3: Configure Prometheus

Open the configuration file:

```bash
nano prometheus.yml
```

Add the following configuration:

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

  # Node Exporter monitoring
  - job_name: "node-exporter"
    static_configs:
      - targets: ["localhost:9100"]
        labels:
          app: "node-exporter"
```

Save the file:

- Press `CTRL + O`
- Press `ENTER`
- Press `CTRL + X`

### Understanding prometheus.yml

**1. global**

```yaml
global:
  scrape_interval: 15s
```

Prometheus collects metrics every 15 seconds.

**2. evaluation_interval**

```yaml
evaluation_interval: 15s
```

Prometheus evaluates configured alerting and recording rules every 15 seconds.

**3. scrape_configs**

This section defines the monitoring targets.

**4. Prometheus job**

```yaml
- job_name: "prometheus"
  static_configs:
    - targets: ["localhost:9090"]
```

Prometheus monitors its own metrics.

**5. Node Exporter job**

```yaml
- job_name: "node-exporter"
  static_configs:
    - targets: ["localhost:9100"]
```

Prometheus collects Linux metrics from Node Exporter.

### Step 4: Validate Prometheus Configuration

Run:

```bash
./promtool check config prometheus.yml
```

Expected result:

```text
SUCCESS: prometheus.yml is valid prometheus config file syntax
```

### Step 5: Start Prometheus

```bash
./prometheus --config.file=prometheus.yml
```

Keep this terminal open.

### Step 6: Verify Prometheus

Open your browser:

http://localhost:9090

Check the listening port:

```bash
sudo ss -lntp | grep 9090
```

Expected output should show Prometheus listening on port `9090`.

### Step 7: Verify Monitoring Targets

Open:

http://localhost:9090/targets

You should see:

| Job | Endpoint | Status |
|---|---|---|
| prometheus | localhost:9090 | UP |
| node-exporter | localhost:9100 | UP |

**What does UP mean?**

Prometheus successfully connected to the target and collected metrics.

If a target shows DOWN, check the target address, port, exporter process and network connectivity.

---

# PART 3 – GRAFANA INSTALLATION AND CONFIGURATION

## 6. What is Grafana?

Grafana is an open-source visualization and analytics platform.

It connects to data sources such as Prometheus and displays collected metrics through dashboards.

### Important Grafana Features

- Custom dashboards
- CPU monitoring
- Memory monitoring
- Disk monitoring
- Network monitoring
- Alerts and notifications
- Dashboard sharing

### Step 1: Verify Grafana Service

Check Grafana status:

```bash
sudo systemctl status grafana-server
```

If it is installed but stopped:

```bash
sudo systemctl start grafana-server
```

Enable it to start automatically when the service manager starts:

```bash
sudo systemctl enable grafana-server
```

Check the port:

```bash
sudo ss -lntp | grep 3000
```

### Step 2: Open Grafana

Open your browser:

http://localhost:3000

Log in using the credentials configured during installation.

For a standard Grafana installation, the initial credentials are commonly:

```text
Username: admin
Password: admin
```

You may be asked to change the password on first login.

### Step 3: Add Prometheus as a Data Source

Follow these steps:

1. Log in to Grafana.
2. Navigate to **Connections**.
3. Select **Data sources**.
4. Click **Add data source**.
5. Select **Prometheus**.

Configure the following:

```text
Name: Prometheus

Prometheus Server URL:
http://localhost:9090
```

Click:

```text
Save & test
```

Expected result:

```text
Successfully queried the Prometheus API.
```

**Important:** Our Grafana and Prometheus are running on the same Ubuntu environment. Therefore, `localhost:9090` works for this setup.

If Grafana runs inside a separate container, its `localhost` refers to that container, not the Ubuntu host. In that situation, use an appropriate reachable service address.

---

# PART 4 – IMPORT LINUX MONITORING DASHBOARD

## 7. Import Node Exporter Full Dashboard

We used the popular community dashboard:

**Dashboard Name:** Node Exporter Full

**Dashboard ID:** 1860

Official dashboard:

https://grafana.com/grafana/dashboards/1860-node-exporter-full/

### Step 1: Import Dashboard

1. Open Grafana.
2. Navigate to **Dashboards**.
3. Click **New**.
4. Select **Import**.
5. Enter dashboard ID:

```text
1860
```

6. Click **Load**.
7. Select your Prometheus data source.
8. Click **Import**.

### Step 2: Verify Dashboard

Your dashboard should display Linux server metrics.

Important monitoring sections:

| Metric | What it tells us |
|---|---|
| CPU Usage | Processor utilization |
| Memory Usage | RAM utilization |
| Disk Usage | Disk space consumption |
| Filesystem | Mounted filesystem information |
| Network Traffic | Incoming and outgoing traffic |
| System Load | System workload |
| Uptime | How long the system has been running |

**Important:** If some panels show no data, check the dashboard's job filter. Our Prometheus job name is `node-exporter`; the dashboard may expect another job label.

---

# PART 5 – PROMQL PRACTICAL

## 8. What is PromQL?

PromQL stands for Prometheus Query Language.

It is used to query and analyze metrics stored in Prometheus.

Open:

http://localhost:9090/graph

Try the following queries.

### 1. Check CPU Metrics

```promql
node_cpu_seconds_total
```

Shows cumulative CPU time for different CPU modes.

### 2. Check Total Memory

```promql
node_memory_MemTotal_bytes
```

Shows total system memory in bytes.

### 3. Check Available Memory

```promql
node_memory_MemAvailable_bytes
```

Shows available memory in bytes.

### 4. Calculate Memory Usage Percentage

```promql
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
```

### 5. Check Filesystem Size

```promql
node_filesystem_size_bytes
```

### 6. Check Filesystem Available Space

```promql
node_filesystem_avail_bytes
```

### 7. Check Network Receive Traffic

```promql
rate(node_network_receive_bytes_total[5m])
```

### 8. Check Network Transmit Traffic

```promql
rate(node_network_transmit_bytes_total[5m])
```

These queries help DevOps Engineers investigate Linux server performance.

---

# PART 6 – TROUBLESHOOTING

## 9. Issue 1: Node Exporter Metrics Not Available

**Problem:**

Node Exporter is not responding.

**Check:**

```bash
sudo ss -lntp | grep 9100
```

Check the endpoint:

```bash
curl http://localhost:9100/metrics
```

**Solution:**

Make sure Node Exporter is running and listening on port `9100`.

---

## 10. Issue 2: Duplicate Filesystem Metrics

**Problem:**

We encountered this error in our WSL environment:

```text
error gathering metrics: 8 error(s) occurred

collected metric "node_filesystem_size_bytes"
was collected before with the same name and label values
```

**Root cause identified:**

The `/run/user` mount point was exposed more than once with different device numbers but identical filesystem metric labels.

**Solution:**

We restarted Node Exporter with this filesystem exclusion:

```bash
./node_exporter --collector.filesystem.mount-points-exclude='^/(dev|proc|run/user($|/)|run/credentials/.+|sys|var/lib/docker/.+)($|/)'
```

Verify the metrics endpoint:

```bash
curl -sS -o /tmp/node-metrics.txt -w "%{http_code}\n" http://localhost:9100/metrics
```

Expected:

```text
200
```

Verify `/run/user` is excluded:

```bash
grep 'mountpoint="/run/user"' /tmp/node-metrics.txt
```

Expected: no output.

---

## 11. Issue 3: Prometheus Target Shows DOWN

**Problem:**

Node Exporter target is showing DOWN in Prometheus.

**Check:**

```bash
sudo ss -lntp | grep 9100
```

Test:

```bash
curl http://localhost:9100/metrics
```

Check the Prometheus configuration:

```bash
cat prometheus.yml
```

Verify the target:

```yaml
targets: ["localhost:9100"]
```

Open:

http://localhost:9090/targets

Check the error shown for the failed target.

---

## 12. Issue 4: Grafana Dashboard Shows No Data

**Problem:**

Dashboard is imported, but graphs are empty.

**Check the following:**

1. Is Node Exporter running?
2. Is Prometheus running?
3. Is the Node Exporter target UP?
4. Is Prometheus added as a Grafana data source?
5. Does the dashboard use the correct Prometheus data source?
6. Does the dashboard's job filter match `node-exporter`?

Test this query in Prometheus:

```promql
up{job="node-exporter"}
```

Expected result:

```text
1
```

A value of `1` means the target is up.

---

# PART 7 – IMPORTANT PORTS

## 13. Monitoring Port Reference

| Tool | Port | Purpose |
|---|---:|---|
| Node Exporter | 9100 | Exposes Linux metrics |
| Prometheus | 9090 | Metrics collection and queries |
| Grafana | 3000 | Dashboards and visualization |

Useful URLs:

| Application | URL |
|---|---|
| Node Exporter | http://localhost:9100/metrics |
| Prometheus | http://localhost:9090 |
| Prometheus Targets | http://localhost:9090/targets |
| Grafana | http://localhost:3000 |

---

# PART 8 – PRODUCTION SCENARIO

## 14. Real-Time Company Scenario

Imagine a company has 20 Linux EC2 instances running production applications.

The DevOps team wants to monitor all 20 servers from one central monitoring system.

### Production Architecture

```text
                   AWS CLOUD
                       |
          +------------+------------+
          |            |            |
          v            v            v
       EC2-01        EC2-02        EC2-20
          |            |            |
          v            v            v
       Node          Node          Node
      Exporter      Exporter      Exporter
          |            |            |
          +------------+------------+
                       |
                       v
                 PROMETHEUS
                 Port 9090
                       |
                       v
                   GRAFANA
                 Port 3000
                       |
                       v
               CENTRAL DASHBOARD
                       |
          +------------+------------+
          |            |            |
          v            v            v
         CPU         Memory         Disk
       Monitoring   Monitoring    Monitoring
```

### How does this work in production?

1. Install Node Exporter on every Linux EC2 instance.
2. Configure security groups to allow Prometheus to access Node Exporter on port `9100`.
3. Install Prometheus on a dedicated monitoring server.
4. Configure all EC2 private IP addresses as Prometheus targets.
5. Connect Grafana to Prometheus.
6. Create dashboards for centralized monitoring.
7. Configure alerting for CPU, memory and disk thresholds.

**Security best practice:** Do not expose Node Exporter, Prometheus or Grafana directly to the public internet. Restrict access through private networking, security groups, authentication and appropriate access controls.

---

# PART 9 – PROJECT VALIDATION CHECKLIST

## 15. Final Validation

- [x] Node Exporter installed.
- [x] Node Exporter running on port 9100.
- [x] Node Exporter metrics endpoint returning HTTP 200.
- [x] Filesystem duplicate metric issue resolved.
- [x] Prometheus installed.
- [x] Prometheus configuration created.
- [x] Prometheus running on port 9090.
- [x] Prometheus target showing UP.
- [x] Grafana installed and running.
- [x] Prometheus connected to Grafana.
- [x] Node Exporter Full dashboard imported.
- [x] Real-time Ubuntu metrics visible in Grafana.

---

# PART 10 – INTERVIEW QUESTIONS

## 16. Important DevOps Interview Questions

**Q1. What is Prometheus?**

Prometheus is an open-source monitoring system used to collect and store time-series metrics from applications and infrastructure.

**Q2. What is Node Exporter?**

Node Exporter collects Linux operating system and hardware metrics and exposes them for Prometheus.

**Q3. Why do we use Grafana?**

We use Grafana to visualize metrics collected by Prometheus through dashboards and graphs.

**Q4. What is Prometheus scraping?**

Scraping is the process of periodically collecting metrics from configured monitoring targets.

**Q5. What is PromQL?**

PromQL is the query language used to retrieve and analyze Prometheus metrics.

**Q6. What is a Prometheus target?**

A target is an endpoint from which Prometheus collects metrics.

**Q7. What does UP mean in Prometheus?**

UP means Prometheus successfully scraped the target.

**Q8. What is the default Node Exporter port?**

9100.

**Q9. What is the default Prometheus port?**

9090.

**Q10. What is the default Grafana port?**

3000.

**Q11. Can Prometheus monitor multiple Linux servers?**

Yes. We can configure multiple Node Exporter endpoints as Prometheus targets.

**Q12. What is a time-series database?**

It stores data points along with timestamps, allowing us to analyze how metrics change over time.

**Q13. How do you troubleshoot a DOWN target?**

I check the exporter process, listening port, network connectivity, target configuration and Prometheus error message.

**Q14. How do you monitor CPU usage in Prometheus?**

We can use CPU metrics exposed by Node Exporter and calculate utilization using PromQL.

**Q15. Can Grafana collect metrics directly from Node Exporter?**

Grafana is primarily a visualization platform. In this setup, Prometheus collects the metrics and Grafana queries Prometheus.

---

## 17. Project Outcome

We successfully implemented a Linux monitoring solution using:

- Node Exporter for Linux metric collection.
- Prometheus for metric scraping and storage.
- Grafana for real-time dashboard visualization.

We also resolved a filesystem duplicate metric issue encountered in our WSL Ubuntu environment.

**Final Result:** We can monitor CPU, memory, filesystem, disk and network metrics from our local Ubuntu environment using Grafana.

---

## 18. Future Enhancements

The next improvements for this monitoring project are:

1. Configure Prometheus alerting rules.
2. Install and configure Alertmanager.
3. Send email alerts for critical issues.
4. Monitor multiple Linux EC2 instances.
5. Build custom Grafana dashboards.
6. Monitor Docker containers.
7. Monitor Kubernetes clusters using Prometheus and Grafana.

**Project Status: Successfully Completed – Linux Monitoring with Prometheus, Node Exporter and Grafana.**
