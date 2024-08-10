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


