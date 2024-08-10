# Observability With OpenTelemetry Instrumentation Libraries, Collector and Kubernetes Operator

__Introduction__:
Deploying an application is

---

### Prerequisites

Prerequisites

I am running this simultaneously on both a Windows PC running WSL2 and an old iMac i converted to Ubuntu Linux.

To follow alongwith this project/tutorial, i have the following installed on my pc:

Docker
Kind
Kubectl
Kubens and Kubectx

### Getting Started

#### Setup Kind Kubernetes Cluster

kind create cluster --name=otel-project --image kindest/node:v1.30.2

#### Deploy cert-manager

In this next step i deploy a cert-manager which is used by OpenTelemetry operator to provision TLS certificates for admission webhooks.

```bash
kubectl apply -f cert-manager/cert-manager.yaml
```

#### Deploy Backend for Observability

My sample app doesnt have a backend so i would be installing the following Prometheus compatible tools to make up my backend:

1. [Grafana Mimir](https://github.com/grafana/mimir): (For Database) Grafana Mimir is an open source software project that provides a scalable long-term storage for Prometheus.
2. [Grafana Loki](https://github.com/grafana/loki): (For Logs) Loki is a horizontally-scalable, highly-available, multi-tenant log aggregation system inspired by Prometheus.
3. [Grafana Tempo](https://github.com/grafana/tempo): (For Traces) Grafana Tempo is an open source, easy-to-use and high-scale distributed tracing backend, deeply integrated with Grafana, Prometheus, and Loki.

From my manifest, im deploying this into the namespace : observability-backend.

```bash
kubectl apply -f backend/backend-01.yaml
```

![alt text](deploy-backend01.png)

**Check namespaces using kubens:**

```bash
kubens
```

![alt text](kubens1.png)
