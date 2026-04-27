# WatchTower — Observability Stack for Microservices

## Overview

WatchTower is a DevOps observability solution developed for **Reyla Logistics**, a last-mile delivery company experiencing recurring system outages and limited operational visibility. The project implements a production-grade observability stack across three core microservices, enabling real-time monitoring, alerting, and structured logging.

**Monitored Services:**

- Order Service
- Tracking Service
- Notification Service

**Core Capabilities:**

- 📊 Metrics collection via Prometheus
- 📈 Visualization via Grafana
- 🚨 Automated alerting for critical failures
- 🪵 Structured logging via Docker

---

## Architecture

```
                ┌──────────────────────┐
                │        Client        │
                └─────────┬────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
┌───────────────┐ ┌───────────────┐ ┌────────────────────┐
│ order-service │ │tracking-serv  │ │ notification-serv  │
│    :3001      │ │    :3002      │ │       :3003        │
└──────┬────────┘ └──────┬────────┘ └────────┬───────────┘
       │                  │                   │
       └──────────┬───────┴──────────┬────────┘
                  │                  │
            ┌──────────────┐   ┌──────────────┐
            │  Prometheus  │──▶│   Grafana    │
            │    :9090     │   │    :3000     │
            └──────────────┘   └──────────────┘
                     │
                     ▼
               🚨 Alerts Engine
```

---

## Setup Instructions

### 1. Clone the Repository

```bash
git clone <your-repo-link>
cd WatchTower/app
```

### 2. Configure Environment Variables

Copy the example environment file and populate the required values:

```bash
cp .env.example .env
```

```env
ORDER_SERVICE_PORT=3001
TRACKING_SERVICE_PORT=3002
NOTIFICATION_SERVICE_PORT=3003
PROMETHEUS_PORT=9090
GRAFANA_PORT=3000
```

### 3. Start the Stack

```bash
docker compose up --build
```

### 4. Verify Services

**Prometheus** — Navigate to `http://localhost:9090/targets` and confirm all services display a status of `UP`.

**Grafana** — Navigate to `http://localhost:3000`. Use the default credentials (`admin` / `admin`). The dashboard loads automatically upon login.

---

## Dashboard Overview

The auto-provisioned Grafana dashboard includes the following panels:

| Panel                 | Description                                                                                      |
| --------------------- | ------------------------------------------------------------------------------------------------ |
| **HTTP Request Rate** | Displays request throughput per service using `sum(rate(order_service_http_requests_total[1m]))` |
| **Error Rate (5xx)**  | Tracks the failure rate across services to detect instability early                              |
| **Service Health**    | Based on the `up` metric — `1` indicates healthy, `0` indicates down                             |

---

## Alerting

Alert rules are defined in `prometheus/alerts.yml`.

### 🔴 ServiceDown — Critical

**Condition:** `up == 0`

**Test procedure:**

```bash
docker compose stop order-service
```

**Result:** Alert fires after 1 minute.

---

### 🟡 ServiceNotScraping — Warning

**Condition:** Prometheus is unable to scrape metrics from a target.

**Test procedure:** Update `prometheus.yml` to point to an invalid port, then restart Prometheus:

```yaml
targets: ["order-service:9999"]
```

```bash
docker compose restart prometheus
```

**Result:** Alert fires after 2 minutes.

---

### 🟡 HighErrorRate — Warning

**Condition:** More than 5% of requests return a `5xx` status code.

**Test procedure:** Inject a simulated failure into the Order Service:

```js
if (Math.random() < 0.3) {
  return res.status(500).json({ error: "Simulated failure" });
}
```

Generate traffic to trigger the alert:

```bash
for i in {1..50}; do \
  curl -X POST http://localhost:3001/orders \
  -H "Content-Type: application/json" \
  -d '{"item":"phone","quantity":2}'; \
done
```

**Result:** Alert triggers after 5 minutes.

---

## Logging

Docker is configured to use JSON-structured logging for all services.

**Stream all service logs:**

```bash
docker compose logs -f
```

**Filter logs by service (e.g., error-level entries for Order Service):**

```bash
docker compose logs order-service | grep error
```

**Example log output:**

```json
{
  "level": "error",
  "service": "order-service",
  "msg": "Failed to notify",
  "error": "connection refused"
}
```

---

## Features Implemented

- Microservices architecture (3 independent services)
- Prometheus metrics scraping with multi-target configuration
- Auto-provisioned Grafana dashboards
- Alerting rules with tiered severity levels (critical / warning)
- Structured JSON logging via Docker
- Full Docker Compose orchestration

---

## Tech Stack

| Layer            | Technology              |
| ---------------- | ----------------------- |
| Runtime          | Node.js                 |
| Framework        | Express.js              |
| Metrics          | Prometheus              |
| Visualization    | Grafana                 |
| Containerization | Docker & Docker Compose |
