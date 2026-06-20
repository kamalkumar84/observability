                    +----------------+
                    |    Grafana     |
                    +-------+--------+
                            |
      +---------------------+---------------------+
      |                     |                     |
      v                     v                     v

+-------------+    +-------------+    +-------------+
| Victoria    |    |    Loki     |    |    Tempo    |
| Metrics     |    |             |    |             |
+------+------+    +------+------+    +------+------+
       ^                    ^                  ^
       |                    |                  |
       +--------------------+------------------+
                            |
                    OTEL Gateway
                            ^
                            |
             +--------------+--------------+
             |                             |
      OTEL Collectors                Exporters
      (all servers)                   MySQL/

1. Create Monitoring Server

AWS Instance:

OS: Ubuntu 24.04 LTS ARM64
Instance: t4g.xlarge
vCPU: 4
RAM: 16 GB
Disk: 250 GB gp3

Security Group:

22     SSH
3000   Grafana
8428   VictoriaMetrics
3100   Loki
3200   Tempo
4317   OTLP gRPC
4318   OTLP HTTP
9093   Alertmanager (optional)


2. Install Docker and compose

install otel collector in ARM linux
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/latest/download/otelcol-contrib_linux_arm64.deb

sudo dpkg -i otelcol-contrib_linux_arm64.deb

For production, I recommend separate Docker Compose files per service. This makes upgrades, troubleshooting, backups, and migrations much easier.

Directory structure:

/opt/monitoring
├── grafana/
├── victoriametrics/
├── loki/
├── tempo/
├── otel-gateway/
└── docker-network/

here we have need data in /data partition

Directory Structure
mkdir -p \
/data/grafana/data \
/data/victoriametrics/data \
/data/loki/data \
/data/loki/config \
/data/tempo/data \
/data/tempo/config \
/data/otel/config \
/data/docker-compose

Store all compose files under:

/opt/monitoring/

and all persistent data under:

/data/

create config for loki

mkdir -p /data/loki/config

Tempo config
mkdir -p /data/tempo/config

Otel collector config

mkdir -p /data/otel/config

Startup Order

cd /opt/monitoring/victoriametrics && docker compose up -d

cd /opt/monitoring/loki && docker compose up -d

cd /opt/monitoring/tempo && docker compose up -d

cd /opt/monitoring/otel-gateway && docker compose up -d

cd /opt/monitoring/grafana && docker compose up -d

verify
docker ps

| Service         | URL                           |
| --------------- | ----------------------------- |
| Grafana         | `http://SERVER_IP:3000`       |
| VictoriaMetrics | `http://SERVER_IP:8428`       |
| Loki            | `http://SERVER_IP:3100/ready` |
| Tempo           | `http://SERVER_IP:3200/ready` |
| OTEL Gateway    | `4317/4318` (OTLP endpoints)  |


