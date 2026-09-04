# DEV-423 — Resource Usage Anomaly Detection

Grafana dashboard for monitoring container CPU utilization and highlighting sustained high CPU usage that may indicate a resource-usage anomaly.

## Scope

This implementation focuses on:

- Current CPU utilization by container
- Detection of sustained CPU utilization above 80%
- Visualization of detected high-CPU anomalies in Grafana

High CPU usage is treated as a **potential resource anomaly**, not as proof of malicious activity or crypto mining.

## Architecture

```text
Docker containers
       │
       ▼
   cAdvisor
       │
       ▼
   Prometheus
       │
       ▼
    Grafana
       │
       ▼
Security Monitoring
└── Resource Usage Anomaly Detection
```

## Components

| Component | Version | Purpose |
|---|---|---|
| Grafana | 13.2.1 | Dashboard and visualization |
| Prometheus | 3.14.0 | Metrics collection and querying |
| cAdvisor | 0.52.1 | Container resource metrics |

## Dashboard

**Dashboard:** Resource Usage Anomaly Detection

**Folder:** Security Monitoring

### Panels

1. **CPU Utilization by Container**
2. **High CPU Anomalies (>80%)**

The CPU utilization panel uses a 5-minute rate of `container_cpu_usage_seconds_total`.

The anomaly panel identifies containers whose 5-minute CPU rate exceeds 80%:

```promql
(
  sum by (name) (
    rate(
      container_cpu_usage_seconds_total{
        job="cadvisor",
        name!=""
      }[5m]
    )
  ) * 100
) > 80
```

Using a 5-minute rate helps avoid treating short CPU spikes as sustained anomalies.

## Local Setup

### Prerequisites

- Docker
- Docker Compose

### Configuration

Create `.env` from `.env.example`:

```bash
cp .env.example .env
```

Set the local Grafana credentials in `.env`.

The `.env` file is intentionally excluded from Git.

### Start the Monitoring Stack

```bash
docker compose up -d
```

Check the containers:

```bash
docker ps --filter name=dev423
```

### Access

**Grafana:** http://localhost:3000

**Prometheus:** http://localhost:9090

The Grafana dashboard is provisioned automatically under the **Security Monitoring** folder.

### Stop the Monitoring Stack

To stop the monitoring stack:

```bash
docker compose down
```

To also remove the local Grafana data volume:

```bash
docker compose down -v
```

## Validation

The monitoring pipeline was validated end-to-end using a temporary synthetic CPU workload.

A temporary Alpine container was used to generate sustained CPU load:

```bash
docker run -d \
  --name dev423-cpu-test \
  alpine:3.22 \
  sh -c 'while true; do :; done'
```

The workload was observed by cAdvisor and Prometheus. The 5-minute anomaly query returned the test container at approximately **91.8% CPU**, and the Grafana anomaly panel highlighted it in red.

After validation, the synthetic workload was removed:

```bash
docker rm -f dev423-cpu-test
```

The anomaly query subsequently returned no active anomalies.

## Reproducibility

The stack uses pinned container image versions rather than `latest`.

The Prometheus datasource uses the deterministic UID:

```text
dev423-prometheus
```

Grafana dashboard provisioning uses the same datasource UID, allowing the dashboard to be recreated consistently from a clean Grafana volume.

## Security Notes

- This is a local isolated test environment.
- No production credentials or production data are required.
- Grafana credentials are supplied through `.env`.
- `.env` is excluded from version control.
- The CPU anomaly detection indicates sustained high resource utilization; it does not independently determine whether activity is malicious.