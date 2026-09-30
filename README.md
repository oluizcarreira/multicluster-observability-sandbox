# Multicluster Observability Sandbox 🚀

### FastAPI | OpenTelemetry | KinD | Tempo | Prometheus | Loki | Grafana

This repository contains a complete implementation of a distributed enterprise observability architecture running locally using **Kubernetes (KinD)**.

The project demonstrates complete separation of contexts between the application environment, the monitoring infrastructure, and the visualization layer, using the industry-standard **OpenTelemetry** framework to collect the three pillars of observability: **Traces, Metrics, and Logs**.

---

## 🏗️ System Architecture — 3-Cluster Topology

The environment was designed to simulate production network security guidelines by isolating services into three distinct local clusters that communicate through port forwarding and NodePorts over the internal Docker network:

```text
[ CLUSTER 1: cluster-app ]      [ CLUSTER 2: cluster-obs ]      [ CLUSTER 3: cluster-grafana ]
┌────────────────────────┐      ┌────────────────────────┐      ┌────────────────────────────┐
│  ┌──────────────────┐  │      │  ┌──────────────────┐  │      │                            │
│  │    FastAPI API   │──┼─────>│  │  OTel Collector  │  │      │                            │
│  └────────┬─────────┘  │      │  └────────┬─────────┘  │      │      ┌──────────────┐      │
│           │            │      │           │            │      │      │              │      │
│  ┌────────▼─────────┐  │      │     ┌─────┼─────┐      │      │      │ Grafana (UI) │      │
│  │    PostgreSQL    │  │      │     │     │     │      │      │      │              │      │
│  └──────────────────┘  │      │ ┌───▼─┐ ┌─▼───┐ ┌───▼──┐      │      └──────▲───────┘      │
│                        │      │ │Tempo│ │Prom │ │Loki │◄───┼──┤             │              │
│  ┌──────────────────┐  │      │ └─────┘ └─────┘ └──────┘      │             │              │
│  │ OTel Host Agent  │──┼─────>│                               │     ┌───────▼────────┐     │
│  └──────────────────┘  │      │                               │     │ Ingress (Nginx)│     │
└────────────────────────┘      └────────────────────────┘      └─────┴───────▲────────┴─────┘
                                                                             │
                                                                      (grafana.local)
```

### 1. Cluster 1: `cluster-app`

**FastAPI Application:** Python API instrumented with the OpenTelemetry SDK.

**PostgreSQL:** Relational database used by the application.

**OTel Agent:** Collector running in DaemonSet/Agent mode to collect host hardware metrics such as CPU, memory, and disk usage.

**NGINX Ingress:** Handles external traffic routing to the API.

### 2. Cluster 2: `cluster-obs` — The Central Data Layer

**OpenTelemetry Collector:** Acts as the central telemetry pipeline. It implements optimization processors such as `memory_limiter`, `batch`, and `resource` for OTLP label injection before forwarding telemetry data to the destination backends.

**Grafana Tempo:** Backend responsible for storing distributed traces.

**Prometheus:** Time-series database responsible for storing metrics.

**Loki:** Specialized log aggregation system for storing and querying logs.

### 3. Cluster 3: `cluster-grafana` — Visualization Layer

**Grafana:** Visualization interface configured through code using ConfigMaps to connect to the `cluster-obs` through NodePorts (`30090`, `30200`, `30100`).

**NGINX Ingress:** Exposes the visualization interface through the custom URL `grafana.local`.

---

## 📂 Project Structure

```text
.
├── app/
│   ├── Dockerfile                  # Application build instructions
│   ├── main.py                     # Instrumented Python API source code
│   └── requirements.txt            # Python dependencies
├── infra/
│   ├── kind-cluster-app.yaml       # Physical definition of Cluster 1 (App)
│   ├── kind-cluster-grafana.yaml   # Physical definition of Cluster 3 (UI - Port 80)
│   └── kind-cluster-obs.yaml       # Physical definition of Cluster 2 (Observability)
└── k8s/
    ├── cluster-app/                # Application manifests
    │   ├── api-deployment.yaml
    │   ├── ingress-api.yaml
    │   ├── otel-agent.yaml
    │   └── postgres.yaml
    ├── cluster-grafana/            # Visualization layer manifests
    │   ├── grafana.yaml
    │   └── ingress-grafana.yaml
    └── cluster-obs/                # Telemetry backend manifests
        ├── ingress-obs.yaml
        ├── loki.yaml
        ├── otel-collector.yaml
        ├── prometheus.yaml
        └── tempo.yaml
```

---

## 🚀 Getting Started

### Prerequisites

Make sure the following tools are installed and available in your `PATH`:

- Docker
- Kubernetes
- KinD (Kubernetes in Docker)
- `kubectl`

Also make sure your operating system's `hosts` file contains the following local entry:

**Windows:**

```text
C:\Windows\System32\drivers\etc\hosts
```

**Linux/macOS:**

```text
/etc/hosts
```

Add:

```text
127.0.0.1 grafana.local
```

---

### Step 1: Create the Infrastructure

Create the three isolated Kubernetes clusters using KinD:

```bash
kind create cluster --config infra/kind-cluster-app.yaml --name cluster-app

kind create cluster --config infra/kind-cluster-obs.yaml --name cluster-obs

kind create cluster --config infra/kind-cluster-grafana.yaml --name cluster-grafana
```

---

### Step 2: Install Ingress Controllers

To support URL-based routing, install the NGINX Ingress Controller on the applicable clusters.

#### Cluster App

```bash
kubectl apply \
  -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml \
  --context kind-cluster-app
```

#### Cluster Grafana

```bash
kubectl apply \
  -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml \
  --context kind-cluster-grafana
```

Wait until the `ingress-nginx` pods are in the **Running** state before proceeding.

---

### Step 3: Build and Deploy the Application

Build the API image and load it into the application cluster:

```bash
cd app

docker build -t minha-api-app:latest .

kind load docker-image minha-api-app:latest --name cluster-app

cd ..
```

Apply the application manifests:

```bash
kubectl apply \
  -f k8s/cluster-app/ \
  --context kind-cluster-app
```

---

### Step 4: Deploy the Observability Backend

Deploy the telemetry ingestion and storage stack to `cluster-obs`:

```bash
kubectl apply \
  -f k8s/cluster-obs/ \
  --context kind-cluster-obs
```

Restart the OpenTelemetry Collector deployment:

```bash
kubectl rollout restart deployment otel-collector \
  --context kind-cluster-obs
```

---

### Step 5: Deploy the Visualization Layer

Deploy Grafana and its routing configuration to `cluster-grafana`:

```bash
kubectl apply \
  -f k8s/cluster-grafana/ \
  --context kind-cluster-grafana
```

---

## 🧪 Validation and Testing

### 1. Generate Telemetry Data

Send continuous traffic to the application to populate the observability pipelines:

```bash
for i in {1..20}; do
  curl -s http://localhost:8080/health
  echo ""
done
```

This generates requests that can be observed through the traces, metrics, and logs pipelines.

---

### 2. Access Grafana

Open your browser and navigate to:

```text
http://grafana.local
```

Default credentials:

```text
Username: admin
Password: admin
```

> **Note:** These are the default credentials configured for this local sandbox environment. They should not be used in a production deployment.

---

### 3. Explore the Three Pillars

Navigate to **Explore** in Grafana.

#### 🔴 Traces — Tempo

Select **Tempo**, switch to the **Search** tab, filter by:

```text
api-app-python
```

Then click **Run Query** to inspect the execution flow of individual requests.

#### 🟢 Metrics — Prometheus

Select **Prometheus** and run queries such as:

```promql
http_server_duration_milliseconds_count
```

to inspect traffic volume, or:

```promql
system_cpu_utilization
```

to inspect host CPU utilization.

#### 🔵 Logs — Loki

Select **Loki** and use the following LogQL query:

```logql
{job="api-app-python"}
```

This allows you to visualize the structured logs emitted directly by the API and correlate them over time with the other telemetry signals.

---

## 🔭 Observability Stack

| Component | Purpose |
|---|---|
| **FastAPI** | Application/API layer |
| **OpenTelemetry SDK** | Application instrumentation |
| **OpenTelemetry Collector** | Telemetry collection and processing |
| **Grafana Tempo** | Distributed trace storage |
| **Prometheus** | Metrics storage and querying |
| **Grafana Loki** | Log aggregation and querying |
| **Grafana** | Visualization and observability interface |
| **NGINX Ingress** | HTTP routing and exposure |
| **PostgreSQL** | Application database |
| **KinD** | Local Kubernetes clusters |

---

## 🎯 Project Goals

This project was created as a hands-on sandbox for studying and demonstrating:

- Distributed observability architectures
- Kubernetes multi-cluster environments
- OpenTelemetry instrumentation
- Distributed tracing
- Metrics collection and monitoring
- Centralized log aggregation
- Grafana dashboards and data sources
- NGINX Ingress
- Kubernetes networking
- Infrastructure separation
- Infrastructure as Code principles
- Local simulation of production-like observability environments

---

## 📜 License

This project is licensed under the **MIT License**.

You are free to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the software, subject to the terms and conditions of the MIT License.

See the [`LICENSE`](LICENSE) file for the complete license text.

---

## 👤 Author

Developed as a technical sandbox for studying **Kubernetes, OpenTelemetry, observability, distributed systems, and cloud-native infrastructure**.
