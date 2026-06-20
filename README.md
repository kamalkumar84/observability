# 🔍 Observability Stack - Complete Monitoring Solution

A production-ready observability platform combining **metrics**, **logs**, and **traces** with OpenTelemetry, VictoriaMetrics, Loki, and Tempo—all visualized through Grafana.

---

## 📊 Architecture Overview

```
                    +----------------+
                    |    Grafana     |
                    |  Visualization |
                    +-------+--------+
                            |
      +---------------------+---------------------+
      |                     |                     |
      v                     v                     v

+-------------+    +-------------+    +-------------+
| Victoria    |    |    Loki     |    |    Tempo    |
| Metrics     |    |   Logs      |    |   Traces    |
+------+------+    +------+------+    +------+------+
       ^                    ^                  ^
       |                    |                  |
       +--------------------+------------------+
                            |
                    OTEL Gateway
                    (Data Aggregator)
                            ^
                            |
             +--------------+--------------+
             |                             |
      OTEL Collectors                Exporters
      (All Servers)              (MySQL/Backend)
```

### 🎯 Component Details

| Component | Purpose | Port |
|-----------|---------|------|
| **Grafana** | Unified visualization & dashboards | 3000 |
| **VictoriaMetrics** | Time-series metrics storage | 8428 |
| **Loki** | Log aggregation & querying | 3100 |
| **Tempo** | Distributed trace storage | 3200 |
| **OTEL Gateway** | Data collection & routing | 4317/4318 |

---

## 🚀 Quick Start

### Prerequisites

- AWS EC2 Instance (or any Linux server)
- Ubuntu 24.04 LTS ARM64+
- Docker & Docker Compose installed
- 250GB+ storage for data

---

## 📋 Step 1: Provision AWS Instance

### Instance Specifications

Create a new EC2 instance with these specifications:

```
OS:       Ubuntu 24.04 LTS ARM64
Instance: t4g.xlarge
vCPU:     4 cores
RAM:      16 GB
Storage:  250 GB (gp3 EBS volume)
```

### Security Group Configuration

Open the following ports for monitoring traffic:

```
Port    | Service            | Protocol | Source
--------|-------------------|----------|----------
22      | SSH                | TCP      | Your IP
3000    | Grafana            | TCP      | 0.0.0.0/0
3100    | Loki               | TCP      | 0.0.0.0/0
3200    | Tempo              | TCP      | 0.0.0.0/0
4317    | OTLP gRPC          | TCP      | 0.0.0.0/0
4318    | OTLP HTTP          | TCP      | 0.0.0.0/0
8428    | VictoriaMetrics    | TCP      | 0.0.0.0/0
9093    | Alertmanager       | TCP      | 0.0.0.0/0 (Optional)
```

---

## 🐳 Step 2: Install Docker & OpenTelemetry Collector

### Install Docker Compose

```bash
# Update package manager
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

### Install OpenTelemetry Collector

For detailed OpenTelemetry Collector installation instructions for your system architecture (ARM64 or x86_64), please refer to **[OTEL_COLLECTOR_SETUP.md](./OTEL_COLLECTOR_SETUP.md)**.

---

## 📁 Step 3: Configure Directory Structure

### Production-Ready Layout

For optimal performance and maintainability, use separate Docker Compose files per service:

```
/opt/monitoring/
├── grafana/
│   └── docker-compose.yml
├── victoriametrics/
│   └── docker-compose.yml
├── loki/
│   ├── docker-compose.yml
│   └── config/
├── tempo/
│   ├── docker-compose.yml
│   └── config/
├── otel-gateway/
│   ├── docker-compose.yml
│   └── config/
└── alertmanager/
    └── docker-compose.yml

/data/
├── grafana/
│   └── data/
├── victoriametrics/
│   └── data/
├── loki/
│   ├── data/
│   └── config/
├── tempo/
│   ├── data/
│   └── config/
└── otel/
    └── config/
```

### Create Directory Structure

```bash
# Create configuration directories
mkdir -p \
  /data/grafana/data \
  /data/victoriametrics/data \
  /data/loki/data \
  /data/loki/config \
  /data/tempo/data \
  /data/tempo/config \
  /data/otel/config

# Create compose file directories
mkdir -p /opt/monitoring/{grafana,victoriametrics,loki,tempo,otel-gateway,alertmanager}

# Set proper permissions
sudo chown -R $USER:$USER /data /opt/monitoring
chmod -R 755 /data /opt/monitoring
```

### Directory Conventions

- **Configuration files**: `/data/*/config/` (versioned, backed up)
- **Persistent data**: `/data/*/data/` (large, not version-controlled)
- **Docker Compose files**: `/opt/monitoring/*/docker-compose.yml` (easy to upgrade)

---

## 🔧 Step 4: Service Configuration

### Create Individual Compose Files

Each service gets its own Docker Compose configuration:

#### VictoriaMetrics (`/opt/monitoring/victoriametrics/docker-compose.yml`)

```yaml
version: '3.8'

services:
  victoriametrics:
    image: victoriametrics/victoria-metrics:latest
    container_name: victoriametrics
    volumes:
      - /data/victoriametrics/data:/victoria-metrics-data
    ports:
      - "8428:8428"
    command:
      - '--storageDataPath=/victoria-metrics-data'
      - '--httpListenAddr=0.0.0.0:8428'
    restart: unless-stopped
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge
```

#### Loki (`/opt/monitoring/loki/docker-compose.yml`)

```yaml
version: '3.8'

services:
  loki:
    image: grafana/loki:latest
    container_name: loki
    volumes:
      - /data/loki/config/loki-config.yml:/etc/loki/local-config.yaml
      - /data/loki/data:/loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge
```

#### Tempo (`/opt/monitoring/tempo/docker-compose.yml`)

```yaml
version: '3.8'

services:
  tempo:
    image: grafana/tempo:latest
    container_name: tempo
    volumes:
      - /data/tempo/config/tempo-config.yml:/etc/tempo/tempo.yml
      - /data/tempo/data:/var/tempo
    ports:
      - "3200:3200"
      - "4317:4317"
      - "4318:4318"
    command: -config.file=/etc/tempo/tempo.yml
    restart: unless-stopped
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge
```

#### OTEL Gateway (`/opt/monitoring/otel-gateway/docker-compose.yml`)

```yaml
version: '3.8'

services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otel-gateway
    volumes:
      - /data/otel/config/otel-collector-config.yml:/etc/otel/config.yaml
    ports:
      - "4317:4317"  # gRPC receiver
      - "4318:4318"  # HTTP receiver
    command: --config=/etc/otel/config.yaml
    restart: unless-stopped
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge
```

#### Grafana (`/opt/monitoring/grafana/docker-compose.yml`)

```yaml
version: '3.8'

services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_USERS_ALLOW_SIGN_UP: 'false'
    volumes:
      - /data/grafana/data:/var/lib/grafana
    ports:
      - "3000:3000"
    restart: unless-stopped
    depends_on:
      - victoriametrics
      - loki
      - tempo
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge
```

---

## 🚀 Step 5: Start Services

### Startup Order (Critical for Dependencies)

Services must start in the correct order to establish data flows:

```bash
# 1. Start metrics storage
cd /opt/monitoring/victoriametrics && docker compose up -d
sleep 10

# 2. Start log aggregation
cd /opt/monitoring/loki && docker compose up -d
sleep 10

# 3. Start trace storage
cd /opt/monitoring/tempo && docker compose up -d
sleep 10

# 4. Start data collection gateway
cd /opt/monitoring/otel-gateway && docker compose up -d
sleep 10

# 5. Start visualization layer
cd /opt/monitoring/grafana && docker compose up -d

# Verify all services are running
docker ps
```

### Expected Output

```
CONTAINER ID   IMAGE                              NAMES
abc123...      victoriametrics/victoria-metrics   victoriametrics
def456...      grafana/loki                       loki
ghi789...      grafana/tempo                      tempo
jkl012...      otel/opentelemetry-collector       otel-gateway
mno345...      grafana/grafana                    grafana
```

---

## ✅ Verification

### Health Check Commands

```bash
# Check all containers
docker ps

# View logs (replace SERVICE_NAME)
docker logs SERVICE_NAME -f

# Test endpoints
curl http://localhost:3000           # Grafana
curl http://localhost:8428           # VictoriaMetrics
curl http://localhost:3100/ready     # Loki health
curl http://localhost:3200/ready     # Tempo health
```

---

## 🌐 Access Services

| Service | URL | Default Credentials |
|---------|-----|-------------------|
| **Grafana** | `http://SERVER_IP:3000` | `admin` / `admin` |
| **VictoriaMetrics** | `http://SERVER_IP:8428` | N/A |
| **Loki API** | `http://SERVER_IP:3100` | N/A |
| **Tempo API** | `http://SERVER_IP:3200` | N/A |
| **OTEL Gateway** | `grpc://SERVER_IP:4317` | N/A |
| | `http://SERVER_IP:4318` | N/A |

Replace `SERVER_IP` with your actual instance IP address.

---

## 📚 Configuration Templates

Create configuration files in the designated directories:

### Loki Config (`/data/loki/config/loki-config.yml`)

```yaml
auth_enabled: false

ingester:
  chunk_idle_period: 3m
  max_chunk_age: 5m
  max_streams_per_user: 0

limits_config:
  enforce_metric_name: false

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

server:
  http_listen_port: 3100
```

### Tempo Config (`/data/tempo/config/tempo-config.yml`)

```yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces
```

---

## 💡 Best Practices

✅ **Separate Compose files** per service for easier management  
✅ **Persistent volumes** on `/data/` partition for performance  
✅ **Configuration versioning** in `/data/*/config/`  
✅ **Health checks** and restart policies  
✅ **Network isolation** via Docker networks  
✅ **Regular backups** of `/data/` directory  
✅ **Resource limits** for production deployments  

---

## 🔄 Common Operations

### View Service Logs

```bash
docker logs -f grafana          # Real-time logs
docker logs grafana --tail 50   # Last 50 lines
```

### Restart a Service

```bash
cd /opt/monitoring/grafana && docker compose restart
```

### Stop All Services

```bash
for service in victoriametrics loki tempo otel-gateway grafana; do
  cd /opt/monitoring/$service && docker compose down
done
```

### Update a Service

```bash
cd /opt/monitoring/grafana && \
docker compose pull && \
docker compose up -d
```

---

## 📖 Additional Resources

- [Grafana Documentation](https://grafana.com/docs/)
- [VictoriaMetrics Docs](https://docs.victoriametrics.com/)
- [Loki Documentation](https://grafana.com/docs/loki/)
- [Tempo Documentation](https://grafana.com/docs/tempo/)
- [OpenTelemetry Guide](https://opentelemetry.io/docs/)

---

## 📝 License

This observability stack configuration is provided as-is for educational and production use.

---

**Last Updated**: June 2024  
**Status**: Production Ready ✅
