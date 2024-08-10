# Observability With OpenTelemetry Instrumentation Libraries, Collector and Kubernetes Operator

__Introduction__:
Deploying an application is

---

### Prerequisites

Prerequisites
This tutorial requires just requieres two major things:

A tool like docker or podman to run OCI Container Images
Access to a Kubernetes cluster within the version 1.19-1.26
In case you do not have access to a K8S cluster, you can use Kind or Minikube for a local Kubernetes cluster installations.

Here you will find our cluster setup.

### Getting Started

#### Deploy cert-manager

We start by deploying a cert-manager which is used by OpenTelemetry operator to provision TLScertificates for admission webhooks.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
```

