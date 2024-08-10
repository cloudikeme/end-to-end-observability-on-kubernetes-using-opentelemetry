# Observability With OpenTelemetry Instrumentation Libraries, Collector and Kubernetes Operator

__Introduction__:
Deploying an application is

---

### Prerequisites

Prerequisites
This tutorial requires just requieres two major things:

Docker
Kind
Kubectl
Kubens and Kubectx

Here you will find our cluster setup.

### Getting Started

#### Setup Kind Kubernetes Cluster

kind create cluster --name=otel-project --image kindest/node:v1.30.2

#### Deploy cert-manager

In this next step i deploy a cert-manager which is used by OpenTelemetry operator to provision TLS certificates for admission webhooks.

```bash
kubectl apply -f cert-manager/cert-manager.yaml
```

#### Deploy Backend for Observability

My sample app doesnt have a backend so i would be installing the following Prometheus compatible backend tools: 

1. [Grafana Tempo](https://github.com/grafana/tempo): Grafana Tempo is an open source, easy-to-use and high-scale distributed tracing backend, deeply integrated with Grafana, Prometheus, and Loki.
2. [Grafana Mimir](https://github.com/grafana/mimir): Grafana Mimir is an open source software project that provides a scalable long-term storage for Prometheus
3. [Grafana Loki](https://github.com/grafana/loki): Loki is a horizontally-scalable, highly-available, multi-tenant log aggregation system inspired by Prometheus


