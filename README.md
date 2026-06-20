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
