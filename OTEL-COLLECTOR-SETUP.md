# 🔧 OpenTelemetry Collector - Complete Setup Guide

Complete guide for installing, configuring, and managing the OpenTelemetry Collector as a system service on Linux.

---

## 📋 Table of Contents

1. [Installation](#-installation)
2. [Configuration](#-configuration)
3. [Service Setup](#-service-setup)
4. [Verification & Troubleshooting](#-verification--troubleshooting)

---

## 📦 Installation

### Prerequisites

- Ubuntu 20.04 LTS or higher
- Root or sudo access
- Internet connectivity to download packages
- 2GB+ available disk space

### Step 1: Download OpenTelemetry Collector

Choose the appropriate version for your architecture:

#### For ARM64 (AWS t4g instances)

```bash
# Create installation directory
sudo mkdir -p /opt/otel
cd /tmp

# Download ARM64 DEB package
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.130.0/otelcol-contrib_0.130.0_linux_arm64.deb

# Install
sudo dpkg -i otelcol-contrib_0.130.0_linux_arm64.deb

# Verify installation
/usr/bin/otelcol-contrib --version
```

#### For x86_64 (Intel/AMD processors)

```bash
# Create installation directory
sudo mkdir -p /opt/otel
cd /tmp

# Download x86_64 TAR package (recommended for more control)
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.130.0/otelcol-contrib_0.130.0_linux_amd64.tar.gz

# Extract to installation directory
tar -xzf otelcol-contrib_0.130.0_linux_amd64.tar.gz
sudo cp otelcol-contrib /opt/otel/

# Verify installation
/opt/otel/otelcol-contrib --version
```

#### Alternative: Using DEB Package (x86_64)

```bash
cd /tmp

# Download x86_64 DEB package
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.130.0/otelcol-contrib_0.130.0_linux_amd64.deb

# Install
sudo dpkg -i otelcol-contrib_0.130.0_linux_amd64.deb

# Verify
/usr/bin/otelcol-contrib --version
```

### Step 2: Create Configuration Directory

```bash
# Create configuration directory
sudo mkdir -p /etc/otel

# Set proper permissions
sudo chown -R otel:otel /etc/otel
sudo chmod -R 755 /etc/otel
```

### Step 3: Create User & Group (if not already created by package)

```bash
# Check if user exists
id otel || sudo useradd -r -s /bin/false otel

# Verify
id otel
```

---

## ⚙️ Configuration

### Configuration Structure

Create separate configuration files for different scenarios:

```
/etc/otel/
├── config.yaml              # Main combined config (default)
├── config-metrics.yaml      # Metrics-only collection
├── config-logs.yaml         # Logs-only collection
├── config-traces.yaml       # Traces-only collection
└── config-full.yaml         # Complete metrics + logs + traces
```

### Configuration 1: Host Metrics Collection (Recommended)

Create `/etc/otel/config.yaml`:

```yaml
# OpenTelemetry Collector Configuration - Host Metrics
# Collects system and application metrics from the local host

receivers:
  hostmetrics:
    collection_interval: 30s
    scrapers:
      cpu:
        metrics:
          system.cpu.utilization:
            enabled: true
      memory:
        metrics:
          system.memory.utilization:
            enabled: true
      disk: {}
      filesystem:
        include_virtual_filesystems: false
        exclude_mount_points:
          match_type: regexp
          mount_points:
            - ^/snap/.*
            - ^/run/.*
            - ^/sys/.*
            - ^/proc/.*
            - ^/dev/.*
            - ^/boot/.*
        exclude_fs_types:
          match_type: strict
          fs_types:
            - squashfs
            - tmpfs
            - devtmpfs
            - overlay
            - nsfs
            - cgroup
            - cgroup2
        metrics:
          system.filesystem.usage:
            enabled: true
          system.filesystem.utilization:
            enabled: true
          system.filesystem.inodes.usage:
            enabled: true
      network: {}
      load: {}
      paging: {}
      processes: {}
      process:
        mute_process_name_error: true
        mute_process_exe_error: true
        mute_process_io_error: true
        scrape_process_delay: 0s
        metrics:
          process.cpu.time:
            enabled: true
          process.cpu.utilization:
            enabled: true
          process.memory.usage:
            enabled: true
          process.memory.virtual:
            enabled: true
          process.disk.io:
            enabled: true
          process.threads:
            enabled: true
          process.open_file_descriptors:
            enabled: true
          process.signals_pending:
            enabled: false
        include:
          names:
            - postgres
            - postmaster
            - pg_wal
            - pg_checkpointer
            - pg_autovacuum
            - node
            - java
            - python
            - python3
            - nginx
            - otelcol
            - otelcol-contrib
          match_type: regexp

processors:
  batch/metrics:
    timeout: 15s
    send_batch_size: 2000
    send_batch_max_size: 2000

  # Detect EC2 instance metadata
  resourcedetection/ec2:
    detectors:
      - 'ec2'
    ec2:
      tags:
        - '^Name$'
        - '^Module$'

  # Detect system information
  resourcedetection/system:
    detectors:
      - 'system'
    system:
      hostname_sources:
        - 'os'

  # Transform to add process details
  transform/process:
    error_mode: ignore
    metric_statements:
      - context: datapoint
        statements:
          - set(attributes["process_name"], resource.attributes["process.executable.name"])
          - set(attributes["process_pid"], resource.attributes["process.pid"])
          - set(attributes["process_owner"], resource.attributes["process.owner"])
          - set(attributes["process_cmd"], resource.attributes["process.command.line"])

  # Add custom resource attributes
  resource/metrics:
    attributes:
      - action: upsert
        key: service.name
        value: db
      - action: upsert
        key: field_tier
        value: database
      - action: upsert
        key: field_platform
        value: Linux-os
      - action: upsert
        key: field_env
        value: prod
      - action: upsert
        key: field_appname
        value: postgres

exporters:
  # Export metrics to VictoriaMetrics
  prometheusremotewrite/victoriametrics:
    endpoint: 'http://localhost:8428/api/v1/write'
    tls:
      insecure: true
    resource_to_telemetry_conversion:
      enabled: true
    external_labels:
      cluster: linux-prod

service:
  pipelines:
    metrics:
      receivers:
        - hostmetrics
      processors:
        - resourcedetection/ec2
        - resourcedetection/system
        - resource/metrics
        - transform/process
        - batch/metrics
      exporters:
        - prometheusremotewrite/victoriametrics
```

### Configuration 2: OTLP Receiver (For External Applications)

Create `/etc/otel/config-otlp.yaml` for receiving data from external applications:

```yaml
# OpenTelemetry Collector - OTLP Gateway Configuration
# Receives metrics/logs/traces from applications via OTLP protocol

receivers:
  # gRPC protocol receiver (port 4317)
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024

  # Detect resource information
  resourcedetection/system:
    detectors: [system, env, docker]

exporters:
  # Export metrics to VictoriaMetrics
  prometheusremotewrite/victoriametrics:
    endpoint: 'http://localhost:8428/api/v1/write'
    tls:
      insecure: true

  # Export logs to Loki
  loki:
    endpoint: 'http://localhost:3100/loki/api/v1/push'
    tls:
      insecure: true

  # Export traces to Tempo
  otlp:
    endpoint: 'localhost:4317'
    tls:
      insecure: true

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch, resourcedetection/system]
      exporters: [prometheusremotewrite/victoriametrics]
    
    logs:
      receivers: [otlp]
      processors: [batch, resourcedetection/system]
      exporters: [loki]
    
    traces:
      receivers: [otlp]
      processors: [batch, resourcedetection/system]
      exporters: [otlp]
```

### Configuration 3: Complete Stack (Metrics + OTLP)

Create `/etc/otel/config-full.yaml`:

```yaml
receivers:
  # Host metrics collection
  hostmetrics:
    collection_interval: 30s
    scrapers:
      cpu: {}
      memory: {}
      disk: {}
      filesystem: {}
      network: {}
      processes: {}

  # External application data (OTLP)
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch/metrics:
    timeout: 15s
    send_batch_size: 2000

  resourcedetection/system:
    detectors: [system]

exporters:
  prometheusremotewrite/victoriametrics:
    endpoint: 'http://localhost:8428/api/v1/write'
    tls:
      insecure: true

service:
  pipelines:
    metrics:
      receivers: [hostmetrics, otlp]
      processors: [batch/metrics, resourcedetection/system]
      exporters: [prometheusremotewrite/victoriametrics]
```

### Set Configuration Permissions

```bash
# Copy configurations to /etc/otel
sudo cp *.yaml /etc/otel/ 2>/dev/null || true

# Set permissions
sudo chown -R otel:otel /etc/otel
sudo chmod 644 /etc/otel/*.yaml
sudo chmod 755 /etc/otel

# Verify
ls -la /etc/otel/
```

---

## 🛠️ Service Setup

### Create Systemd Service File

Create `/etc/systemd/system/otelcol.service`:

```ini
[Unit]
Description=OpenTelemetry Collector
Documentation=https://opentelemetry.io/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=otel
Group=otel

# Path to the collector binary
ExecStart=/opt/otel/otelcol-contrib \
  --config=/etc/otel/config.yaml

# Restart policy
Restart=always
RestartSec=5
StartLimitInterval=0

# Process management
KillMode=process
KillSignal=SIGTERM
TimeoutStopSec=30

# Resource limits
LimitNOFILE=65535
LimitNPROC=4096

# Security
NoNewPrivileges=true
PrivateTmp=true

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=otelcol

[Install]
WantedBy=multi-user.target
```

### Alternative Service File (Using /usr/bin path)

If installed via DEB package:

```ini
[Unit]
Description=OpenTelemetry Collector
Documentation=https://opentelemetry.io/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=otel
Group=otel

ExecStart=/usr/bin/otelcol-contrib \
  --config=/etc/otel/config.yaml

Restart=always
RestartSec=5

LimitNOFILE=65535

StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### Enable and Start Service

```bash
# Reload systemd daemon
sudo systemctl daemon-reload

# Enable service to start on boot
sudo systemctl enable otelcol

# Start the service
sudo systemctl start otelcol

# Check status
sudo systemctl status otelcol

# View real-time logs
journalctl -u otelcol -f
```

---

## ✅ Verification & Troubleshooting

### Verify Service is Running

```bash
# Check service status
sudo systemctl status otelcol

# Check if process is running
ps aux | grep otelcol

# View service logs
journalctl -u otelcol -n 50

# View last 100 lines of logs
journalctl -u otelcol --since "1 hour ago" -n 100
```

### Expected Log Output

```
otelcol[12345]: 2024-06-20 10:15:30 INFO loggingexporter/logging_exporter.go:42 MetricsExporter started
otelcol[12345]: 2024-06-20 10:15:30 INFO service/service.go:127 Everything is ready. Begin running and processing data.
```

### Test Metrics Collection

```bash
# Check if metrics are being exported
curl -s http://localhost:8428/api/v1/query?query=up | jq .

# View available metrics
curl -s http://localhost:8428/api/v1/label/__name__/values | jq .
```

### Common Issues & Solutions

#### Issue 1: Permission Denied Error

```bash
# Error: open /etc/otel/config.yaml: permission denied

# Solution:
sudo chown otel:otel /etc/otel/config.yaml
sudo chmod 644 /etc/otel/config.yaml
```

#### Issue 2: Port Already in Use

```bash
# Error: bind: address already in use

# Find process using port 4317:
sudo lsof -i :4317
sudo lsof -i :4318

# Kill the process or change port in config.yaml
```

#### Issue 3: Failed to Export Metrics

```bash
# Error: failed to export metrics to VictoriaMetrics

# Check VictoriaMetrics is running:
curl -s http://localhost:8428/health

# Verify endpoint in config.yaml is correct
grep "endpoint:" /etc/otel/config.yaml
```

#### Issue 4: Service Won't Start

```bash
# Check for configuration syntax errors
/opt/otel/otelcol-contrib --config=/etc/otel/config.yaml --dry-run

# View detailed service logs
journalctl -u otelcol -e

# Check service file syntax
sudo systemd-analyze verify otelcol
```

### Advanced Troubleshooting

```bash
# Run collector in foreground with debug logging
sudo -u otel /opt/otel/otelcol-contrib \
  --config=/etc/otel/config.yaml \
  --set=logging.loglevel=debug

# Validate configuration
/opt/otel/otelcol-contrib \
  --config=/etc/otel/config.yaml \
  validate

# Show available receivers
/opt/otel/otelcol-contrib components | grep receiver

# Show available exporters
/opt/otel/otelcol-contrib components | grep exporter
```

---

## 🔄 Service Management Commands

### Start/Stop/Restart Service

```bash
# Start service
sudo systemctl start otelcol

# Stop service
sudo systemctl stop otelcol

# Restart service
sudo systemctl restart otelcol

# Check status
sudo systemctl status otelcol

# Enable on boot
sudo systemctl enable otelcol

# Disable on boot
sudo systemctl disable otelcol
```

### View and Monitor Logs

```bash
# Real-time logs
sudo journalctl -u otelcol -f

# Last 50 lines
sudo journalctl -u otelcol -n 50

# Logs from last hour
sudo journalctl -u otelcol --since "1 hour ago"

# Logs with severity levels
sudo journalctl -u otelcol -p err -n 100

# Save logs to file
sudo journalctl -u otelcol > otelcol_logs.txt
```

### Update Collector

```bash
# Stop service
sudo systemctl stop otelcol

# Download and install new version
cd /tmp
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.131.0/otelcol-contrib_0.131.0_linux_arm64.deb
sudo dpkg -i otelcol-contrib_0.131.0_linux_arm64.deb

# Or for TAR installation:
# tar -xzf otelcol-contrib_0.131.0_linux_arm64.tar.gz
# sudo cp otelcol-contrib /opt/otel/

# Start service
sudo systemctl start otelcol

# Verify
sudo systemctl status otelcol
```

### Rotate Configuration Without Restart

For hot-reload capability (requires configurable feature):

```bash
# Edit configuration
sudo vi /etc/otel/config.yaml

# Some versions support reload (send HUP signal)
sudo systemctl reload otelcol

# Otherwise restart is required
sudo systemctl restart otelcol
```

---

## 📊 Performance Tuning

### Increase Resource Limits

Edit `/etc/systemd/system/otelcol.service`:

```ini
[Service]
# Increase file descriptor limit
LimitNOFILE=131072
# Increase process limit
LimitNPROC=8192
# Increase memory limit (if needed)
MemoryLimit=4G
```

Then reload:

```bash
sudo systemctl daemon-reload
sudo systemctl restart otelcol
```

### Configure Batch Processor Settings

Adjust in `/etc/otel/config.yaml`:

```yaml
processors:
  batch/metrics:
    # Time after which to send batch
    timeout: 15s
    # Send batch when this many items collected
    send_batch_size: 2000
    # Maximum batch size
    send_batch_max_size: 4000
```

### Memory Management

Monitor memory usage:

```bash
# Real-time memory monitoring
watch -n 1 'ps aux | grep otelcol'

# Memory details
ps -o pid,vsz,rss,comm -p $(pgrep otelcol)
```

---

## 📚 References

- [OpenTelemetry Collector Documentation](https://opentelemetry.io/docs/collector/)
- [OTLP Receiver Configuration](https://github.com/open-telemetry/opentelemetry-collector/blob/main/receiver/otlpreceiver/README.md)
- [Host Metrics Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/hostmetricsreceiver/README.md)
- [Systemd Service Files](https://www.freedesktop.org/software/systemd/man/systemd.service.html)

---

**Last Updated**: June 2024  
**Status**: Production Ready ✅
