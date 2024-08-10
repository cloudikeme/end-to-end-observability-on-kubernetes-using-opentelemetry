# End-to-End Observability With OpenTelemetry Instrumentation Libraries, Collector and Kubernetes Operator

__Introduction__:
Deploying an application is

---

### Prerequisites

I am running this simultaneously on both a Windows PC running WSL2 and an old iMac i converted to Ubuntu Linux.

To follow along with this project/tutorial, i have the following installed on my PC to get started:

* [Docker]()
* [Kind]()
* [Kubectl]()
* [Kubens]() and [Kubectx]()

### Getting Started

#### Setup Kind Kubernetes Cluster

I will name this cluster otel-project

```bash
kind create cluster --name=otel-project --image kindest/node:v1.30.2
```

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

**From my manifest, im deploying this into the namespace : observability-backend.**

```yaml
# send everything to http://otel-collector.observability-backend.svc.cluster.local:4317
apiVersion: v1
kind: Namespace
metadata:
  name: observability-backend
---
apiVersion: v1
data:
  loki.yaml: YXV0aF9lbmFibGVkOiBmYWxzZQpjaHVua19zdG9yZV9jb25maWc6CiAgICBtYXhfbG9va19iYWNrX3BlcmlvZDogMAppbmdlc3RlcjoKICAgIGNodW5rX2Jsb2NrX3NpemU6IDI2MjE0NAogICAgY2h1bmtfaWRsZV9wZXJpb2Q6IDNtCiAgICBjaHVua19yZXRhaW5fcGVyaW9kOiAxbQogICAgbGlmZWN5Y2xlcjoKICAgICAgICByaW5nOgogICAgICAgICAgICBrdnN0b3JlOgogICAgICAgICAgICAgICAgc3RvcmU6IGlubWVtb3J5CiAgICAgICAgICAgIHJlcGxpY2F0aW9uX2ZhY3RvcjogMQpsaW1pdHNfY29uZmlnOgogICAgZW5mb3JjZV9tZXRyaWNfbmFtZTogZmFsc2UKICAgIHJlamVjdF9vbGRfc2FtcGxlczogdHJ1ZQogICAgcmVqZWN0X29sZF9zYW1wbGVzX21heF9hZ2U6IDE2OGgKc2NoZW1hX2NvbmZpZzoKICAgIGNvbmZpZ3M6CiAgICAgICAgLSBmcm9tOiAiMjAxOC0wNC0xNSIKICAgICAgICAgIGluZGV4OgogICAgICAgICAgICBwZXJpb2Q6IDE2OGgKICAgICAgICAgICAgcHJlZml4OiBpbmRleF8KICAgICAgICAgIG9iamVjdF9zdG9yZTogZmlsZXN5c3RlbQogICAgICAgICAgc2NoZW1hOiB2OQogICAgICAgICAgc3RvcmU6IGJvbHRkYgpzZXJ2ZXI6CiAgICBodHRwX2xpc3Rlbl9wb3J0OiAzMTAwCnN0b3JhZ2VfY29uZmlnOgogICAgYm9sdGRiOgogICAgICAgIGRpcmVjdG9yeTogL2RhdGEvbG9raS9pbmRleAogICAgZmlsZXN5c3RlbToKICAgICAgICBkaXJlY3Rvcnk6IC9kYXRhL2xva2kvY2h1bmtzCnRhYmxlX21hbmFnZXI6CiAgICByZXRlbnRpb25fZGVsZXRlc19lbmFibGVkOiBmYWxzZQogICAgcmV0ZW50aW9uX3BlcmlvZDogMAo=
kind: Secret
metadata:
  name: loki-config
  namespace: observability-backend
type: Opaque
---
apiVersion: v1
kind: Service
metadata:
  labels:
    name: loki
  name: loki
  namespace: observability-backend
spec:
  ports:
  - name: loki-http-metrics
    port: 3100
    targetPort: 3100
  selector:
    name: loki
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: loki
  namespace: observability-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      name: loki
  serviceName: loki
  template:
    metadata:
      labels:
        name: loki
    spec:
      containers:
      - args:
        - -config.file=/etc/loki/loki.yaml
        image: grafana/loki:2.0.0
        imagePullPolicy: IfNotPresent
        livenessProbe:
          httpGet:
            path: /ready
            port: http-metrics
          initialDelaySeconds: 45
        name: loki
        ports:
        - containerPort: 3100
          name: http-metrics
        readinessProbe:
          httpGet:
            path: /ready
            port: http-metrics
          initialDelaySeconds: 45
        volumeMounts:
        - mountPath: /etc/loki
          name: loki-config
        - mountPath: /data
          name: loki-data
      securityContext:
        fsGroup: 10001
        runAsGroup: 10001
        runAsNonRoot: true
        runAsUser: 10001
      volumes:
      - name: loki-data
        persistentVolumeClaim:
          claimName: loki-data
      - name: loki-config
        secret:
          secretName: loki-config
  updateStrategy:
    type: RollingUpdate
  volumeClaimTemplates:
  - apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: loki-data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
data:
  overrides.yaml: |
    overrides:
  tempo.yaml: |
    auth_enabled: false
    compactor:
        compaction:
            compacted_block_retention: 24h
    distributor:
        receivers:
            jaeger:
                protocols:
                    thrift_compact:
                        endpoint: 0.0.0.0:6831
            otlp:
                protocols:
                    grpc:
                        endpoint: 0.0.0.0:55680
    ingester: {}
    server:
        http_listen_port: 3200
    storage:
        trace:
            backend: local
            blocklist_poll: 30s
            local:
                path: /tmp/tempo/traces
            wal:
                path: /var/tempo/wal
kind: ConfigMap
metadata:
  name: tempo
  namespace: observability-backend
---
apiVersion: v1
kind: Service
metadata:
  labels:
    name: tempo
  name: tempo
  namespace: observability-backend
spec:
  ports:
  - name: tempo-prom-metrics
    port: 3200
    targetPort: 3200
  - name: tempo-otlp
    port: 55680
    protocol: TCP
    targetPort: 55680
  - name: http
    port: 80
    targetPort: 3200
  - name: receiver
    port: 6831
    protocol: UDP
    targetPort: 6831
  selector:
    app: tempo
    name: tempo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tempo
  namespace: observability-backend
spec:
  minReadySeconds: 10
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: tempo
      name: tempo
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  template:
    metadata:
      annotations:
        config_hash: 7f4b5fad0e6364b6a2a5ea380281cb0e
      labels:
        app: tempo
        name: tempo
    spec:
      containers:
      - args:
        - -config.file=/conf/tempo.yaml
        - -mem-ballast-size-mbs=1024
        env:
        - name: JAEGER_AGENT_PORT
          value: ""
        image: grafana/tempo:main-e6394c3
        imagePullPolicy: IfNotPresent
        name: tempo
        ports:
        - containerPort: 3200
          name: prom-metrics
        - containerPort: 55680
          name: otlp
          protocol: TCP
        volumeMounts:
        - mountPath: /conf
          name: tempo-conf
      volumes:
      - configMap:
          name: tempo
        name: tempo-conf
---
apiVersion: v1
data:
  mimir.yaml: |
    blocks_storage:
        storage_prefix: blocks
        tsdb:
            dir: /data/tsdb
    common:
        storage:
            filesystem:
                dir: /data
    ingester:
        ring:
            replication_factor: 1
    multitenancy_enabled: false
    server:
        grpc_listen_port: 9095
        http_listen_port: 8080
kind: ConfigMap
metadata:
  name: mimir
  namespace: observability-backend
---
apiVersion: v1
kind: Service
metadata:
  labels:
    name: mimir
  name: mimir
  namespace: observability-backend
spec:
  ports:
  - name: mimir-prom-metrics
    port: 8080
    targetPort: 8080
  - name: http
    port: 80
    targetPort: 8080
  - name: grpc
    port: 9095
    targetPort: 9095
  selector:
    app: mimir
    name: mimir
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mimir
  namespace: observability-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mimir
      name: mimir
  serviceName: mimir
  template:
    metadata:
      annotations:
        config_hash: 379ac24872c3157214f0a463540dd3b5
      labels:
        app: mimir
        name: mimir
    spec:
      containers:
      - args:
        - -config.file=/conf/mimir.yaml
        image: grafana/mimir:2.7.1
        imagePullPolicy: IfNotPresent
        livenessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 1
        name: mimir
        ports:
        - containerPort: 8080
          name: prom-metrics
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 1
        volumeMounts:
        - mountPath: /conf
          name: mimir-conf
        - mountPath: /data
          name: mimir-data
      terminationGracePeriodSeconds: 60
      volumes:
      - configMap:
          name: mimir
        name: mimir-conf
      - name: mimir-data
        persistentVolumeClaim:
          claimName: mimir-data
  updateStrategy:
    type: RollingUpdate
  volumeClaimTemplates:
  - apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: mimir-data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
---
apiVersion: v1
data:
  demo-overview-.json: '{}'
kind: ConfigMap
metadata:
  labels: {}
  name: dashboards-demo-0
  namespace: observability-backend
---
apiVersion: v1
data:
  grafana.ini: |
    [analytics]
    reporting_enabled = false
    [auth.anonymous]
    enabled = true
    org_role = Admin
    [feature_toggles]
    enable = traceToLogs
    [log.frontend]
    enabled = true
    [server]
    http_port = 3000
    root_url = http://localhost:3000/grafana
    serve_from_sub_path = true
    router_logging = true
    [users]
    default_theme = light
kind: ConfigMap
metadata:
  name: grafana-config
  namespace: observability-backend
---
apiVersion: v1
data:
  dashboards.yml: |
    apiVersion: 1
    providers:
        - disableDeletion: true
          editable: false
          folder: ""
          name: dashboards
          options:
            path: /grafana/dashboards
          orgId: 1
          type: file
        - disableDeletion: true
          editable: false
          folder: DEMO
          name: dashboards-demo
          options:
            path: /grafana/dashboards-demo
          orgId: 1
          type: file
kind: ConfigMap
metadata:
  name: grafana-dashboard-provisioning
  namespace: observability-backend
---
apiVersion: v1
data:
  cluster.json: |
    {
      "__inputs": [],
      "__requires": [],
      "annotations": {
        "list": []
      },
      "editable": false,
      "gnetId": null,
      "graphTooltip": 0,
      "hideControls": false,
      "id": null,
      "links": [],
      "panels": [
        {
          "aliasColors": {},
          "bars": false,
          "dashLength": 10,
          "dashes": false,
          "datasource": "TestData",
          "fill": 1,
          "gridPos": {
            "h": 8,
            "w": 24,
            "x": 0,
            "y": 0
          },
          "id": 2,
          "legend": {
            "alignAsTable": false,
            "avg": false,
            "current": false,
            "max": false,
            "min": false,
            "rightSide": false,
            "show": true,
            "total": false,
            "values": false
          },
          "lines": true,
          "linewidth": 1,
          "links": [],
          "nullPointMode": "null",
          "percentage": false,
          "pointradius": 5,
          "points": false,
          "renderer": "flot",
          "repeat": null,
          "seriesOverrides": [],
          "spaceLength": 10,
          "stack": false,
          "steppedLine": false,
          "targets": [],
          "thresholds": [],
          "timeFrom": null,
          "timeShift": null,
          "title": "CPU Usage",
          "tooltip": {
            "shared": true,
            "sort": 0,
            "value_type": "individual"
          },
          "type": "graph",
          "xaxis": {
            "buckets": null,
            "mode": "time",
            "name": null,
            "show": true,
            "values": []
          },
          "yaxes": [
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            },
            {
              "format": "short",
              "label": null,
              "logBase": 1,
              "max": null,
              "min": null,
              "show": true
            }
          ]
        }
      ],
      "refresh": "",
      "rows": [],
      "schemaVersion": 16,
      "style": "dark",
      "tags": ["kubernetes"],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-6h",
        "to": "now"
      },
      "timepicker": {
        "refresh_intervals": ["5s", "10s", "30s", "1m", "5m", "15m", "30m", "1h", "2h", "1d"],
        "time_options": ["5m", "15m", "1h", "6h", "12h", "24h", "2d", "7d", "30d"]
      },
      "timezone": "browser",
      "title": "Cluster",
      "version": 0
    }
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: observability-backend
---
apiVersion: v1
data:
  apps.json: |
    {
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": {
              "type": "grafana",
              "uid": "-- Grafana --"
            },
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "target": {
              "limit": 100,
              "matchAny": false,
              "tags": [],
              "type": "dashboard"
            },
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "fiscalYearStartMonth": 0,
      "graphTooltip": 1,
      "links": [],
      "liveNow": false,
      "panels": [
        {
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 0
          },
          "id": 17,
          "panels": [],
          "title": "Frontend",
          "type": "row"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 1
          },
          "id": 13,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": false
            },
            "tooltip": {
              "mode": "single",
              "sort": "none"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "rate(app_games_total[5m])",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "Games",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                }
              },
              "mappings": []
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 1
          },
          "id": 15,
          "options": {
            "legend": {
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "pieType": "pie",
            "reduceOptions": {
              "calcs": [
                "lastNotNull"
              ],
              "fields": "",
              "values": false
            },
            "tooltip": {
              "mode": "single",
              "sort": "none"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(app_winner) (increase(app_wins_total[$__range]))",
              "legendFormat": "{{app_winner}}",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "Wins Per Player",
          "type": "piechart"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 0,
            "y": 9
          },
          "id": 19,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(http_status_code, net_peer_name) (rate(http_client_duration_count[5m]))",
              "legendFormat": "{{net_peer_name}} - code {{http_status_code}}",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "HTTP Client",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 8,
            "y": 9
          },
          "id": 21,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "single",
              "sort": "none"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(http_status_code) (rate(http_server_duration_count{job=\"frontend-deployment\"}[5m]))",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "HTTP Server Responses",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              },
              "unit": "ms"
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 16,
            "y": 9
          },
          "id": 27,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.5, sum by(job, le) (rate(http_server_duration_bucket{job=\"frontend-deployment\"}[5m])))",
              "legendFormat": "50th percentile",
              "range": true,
              "refId": "A"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.95, sum by(job, le) (rate(http_server_duration_bucket{job=\"frontend-deployment\"}[5m])))",
              "hide": false,
              "legendFormat": "95th percentile",
              "range": true,
              "refId": "B"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.99, sum by(job, le) (rate(http_server_duration_bucket{job=\"frontend-deployment\"}[5m])))",
              "hide": false,
              "legendFormat": "99th percentile",
              "range": true,
              "refId": "C"
            }
          ],
          "title": "Server Request Duration",
          "type": "timeseries"
        },
        {
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 17
          },
          "id": 25,
          "panels": [],
          "title": "Backend1",
          "type": "row"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 18
          },
          "id": 31,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "single",
              "sort": "none"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(http_status_code) (rate(http_server_duration_count{job=\"backend1-deployment\"}[5m]))",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "HTTP Server Responses",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              },
              "unit": "ms"
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 18
          },
          "id": 28,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.5, sum by(job, le) (rate(http_server_duration_bucket{job=\"backend1-deployment\"}[5m])))",
              "legendFormat": "50th percentile",
              "range": true,
              "refId": "A"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.95, sum by(job, le) (rate(http_server_duration_bucket{job=\"backend1-deployment\"}[5m])))",
              "hide": false,
              "legendFormat": "95th percentile",
              "range": true,
              "refId": "B"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.99, sum by(job, le) (rate(http_server_duration_bucket{job=\"backend1-deployment\"}[5m])))",
              "hide": false,
              "legendFormat": "99th percentile",
              "range": true,
              "refId": "C"
            }
          ],
          "title": "Server Request Duration",
          "type": "timeseries"
        },
        {
          "collapsed": true,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 26
          },
          "id": 5,
          "panels": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "fieldConfig": {
                "defaults": {
                  "color": {
                    "mode": "palette-classic"
                  },
                  "custom": {
                    "axisCenteredZero": false,
                    "axisColorMode": "text",
                    "axisLabel": "",
                    "axisPlacement": "auto",
                    "barAlignment": 0,
                    "drawStyle": "line",
                    "fillOpacity": 0,
                    "gradientMode": "none",
                    "hideFrom": {
                      "legend": false,
                      "tooltip": false,
                      "viz": false
                    },
                    "lineInterpolation": "linear",
                    "lineWidth": 1,
                    "pointSize": 5,
                    "scaleDistribution": {
                      "type": "linear"
                    },
                    "showPoints": "auto",
                    "spanNulls": false,
                    "stacking": {
                      "group": "A",
                      "mode": "none"
                    },
                    "thresholdsStyle": {
                      "mode": "off"
                    }
                  },
                  "mappings": [],
                  "thresholds": {
                    "mode": "absolute",
                    "steps": [
                      {
                        "color": "green",
                        "value": null
                      },
                      {
                        "color": "red",
                        "value": 80
                      }
                    ]
                  }
                },
                "overrides": []
              },
              "gridPos": {
                "h": 9,
                "w": 8,
                "x": 0,
                "y": 75
              },
              "id": 11,
              "options": {
                "legend": {
                  "calcs": [],
                  "displayMode": "list",
                  "placement": "bottom",
                  "showLegend": false
                },
                "tooltip": {
                  "mode": "single",
                  "sort": "none"
                }
              },
              "targets": [
                {
                  "datasource": {
                    "type": "prometheus",
                    "uid": "PA58DA793C7250F1B"
                  },
                  "editorMode": "builder",
                  "expr": "sum(rate(dice_roll_count_total[$__rate_interval]))",
                  "legendFormat": "__auto",
                  "range": true,
                  "refId": "A"
                }
              ],
              "title": "Dice Rolls",
              "type": "timeseries"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "fieldConfig": {
                "defaults": {
                  "color": {
                    "mode": "palette-classic"
                  },
                  "custom": {
                    "axisCenteredZero": false,
                    "axisColorMode": "text",
                    "axisLabel": "",
                    "axisPlacement": "auto",
                    "barAlignment": 0,
                    "drawStyle": "bars",
                    "fillOpacity": 0,
                    "gradientMode": "none",
                    "hideFrom": {
                      "legend": false,
                      "tooltip": false,
                      "viz": false
                    },
                    "lineInterpolation": "linear",
                    "lineWidth": 1,
                    "pointSize": 5,
                    "scaleDistribution": {
                      "type": "linear"
                    },
                    "showPoints": "auto",
                    "spanNulls": false,
                    "stacking": {
                      "group": "A",
                      "mode": "percent"
                    },
                    "thresholdsStyle": {
                      "mode": "off"
                    }
                  },
                  "mappings": [],
                  "thresholds": {
                    "mode": "absolute",
                    "steps": [
                      {
                        "color": "green",
                        "value": null
                      },
                      {
                        "color": "red",
                        "value": 80
                      }
                    ]
                  }
                },
                "overrides": []
              },
              "gridPos": {
                "h": 9,
                "w": 8,
                "x": 8,
                "y": 75
              },
              "id": 2,
              "options": {
                "legend": {
                  "calcs": [
                    "lastNotNull",
                    "min",
                    "max",
                    "mean"
                  ],
                  "displayMode": "table",
                  "placement": "bottom",
                  "showLegend": true,
                  "sortBy": "Mean",
                  "sortDesc": true
                },
                "tooltip": {
                  "mode": "multi",
                  "sort": "desc"
                }
              },
              "targets": [
                {
                  "datasource": {
                    "type": "prometheus",
                    "uid": "PA58DA793C7250F1B"
                  },
                  "editorMode": "builder",
                  "expr": "sum by(number) (rate(dice_numbers_count_total[$__rate_interval]))",
                  "hide": false,
                  "legendFormat": "__auto",
                  "range": true,
                  "refId": "A"
                }
              ],
              "title": "Dice Numbers Count",
              "type": "timeseries"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "fieldConfig": {
                "defaults": {
                  "color": {
                    "mode": "continuous-GrYlRd"
                  },
                  "mappings": [],
                  "thresholds": {
                    "mode": "absolute",
                    "steps": [
                      {
                        "color": "green",
                        "value": null
                      },
                      {
                        "color": "red",
                        "value": 80
                      }
                    ]
                  }
                },
                "overrides": []
              },
              "gridPos": {
                "h": 9,
                "w": 8,
                "x": 16,
                "y": 75
              },
              "id": 3,
              "options": {
                "displayMode": "basic",
                "minVizHeight": 10,
                "minVizWidth": 0,
                "orientation": "horizontal",
                "reduceOptions": {
                  "calcs": [
                    "lastNotNull"
                  ],
                  "fields": "",
                  "values": false
                },
                "showUnfilled": true
              },
              "pluginVersion": "9.2.3",
              "targets": [
                {
                  "datasource": {
                    "type": "prometheus",
                    "uid": "PA58DA793C7250F1B"
                  },
                  "editorMode": "builder",
                  "expr": "sum by(number) (increase(dice_numbers_count_total[$__range]))",
                  "hide": false,
                  "legendFormat": "__auto",
                  "range": true,
                  "refId": "A"
                }
              ],
              "title": "Dice Numbers",
              "type": "bargauge"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "fieldConfig": {
                "defaults": {
                  "color": {
                    "mode": "palette-classic"
                  },
                  "custom": {
                    "axisCenteredZero": false,
                    "axisColorMode": "text",
                    "axisLabel": "",
                    "axisPlacement": "auto",
                    "barAlignment": 0,
                    "drawStyle": "line",
                    "fillOpacity": 0,
                    "gradientMode": "none",
                    "hideFrom": {
                      "legend": false,
                      "tooltip": false,
                      "viz": false
                    },
                    "lineInterpolation": "linear",
                    "lineWidth": 1,
                    "pointSize": 5,
                    "scaleDistribution": {
                      "type": "linear"
                    },
                    "showPoints": "auto",
                    "spanNulls": false,
                    "stacking": {
                      "group": "A",
                      "mode": "none"
                    },
                    "thresholdsStyle": {
                      "mode": "off"
                    }
                  },
                  "mappings": [],
                  "thresholds": {
                    "mode": "absolute",
                    "steps": [
                      {
                        "color": "green",
                        "value": null
                      },
                      {
                        "color": "red",
                        "value": 80
                      }
                    ]
                  }
                },
                "overrides": []
              },
              "gridPos": {
                "h": 8,
                "w": 12,
                "x": 0,
                "y": 84
              },
              "id": 7,
              "options": {
                "legend": {
                  "calcs": [],
                  "displayMode": "list",
                  "placement": "bottom",
                  "showLegend": true
                },
                "tooltip": {
                  "mode": "single",
                  "sort": "none"
                }
              },
              "targets": [
                {
                  "datasource": {
                    "type": "prometheus",
                    "uid": "PA58DA793C7250F1B"
                  },
                  "editorMode": "builder",
                  "expr": "sum by(generation) (rate(python_gc_collections_total[$__rate_interval]))",
                  "legendFormat": "generation {{generation}}",
                  "range": true,
                  "refId": "A"
                }
              ],
              "title": "Python Garbage Collection",
              "type": "timeseries"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "description": "This assumes one core - to adjust, modify the \"Divide by scalar\" value to the number of cores you're running",
              "fieldConfig": {
                "defaults": {
                  "color": {
                    "mode": "palette-classic"
                  },
                  "custom": {
                    "axisCenteredZero": false,
                    "axisColorMode": "text",
                    "axisLabel": "",
                    "axisPlacement": "auto",
                    "barAlignment": 0,
                    "drawStyle": "line",
                    "fillOpacity": 0,
                    "gradientMode": "none",
                    "hideFrom": {
                      "legend": false,
                      "tooltip": false,
                      "viz": false
                    },
                    "lineInterpolation": "linear",
                    "lineWidth": 1,
                    "pointSize": 5,
                    "scaleDistribution": {
                      "type": "linear"
                    },
                    "showPoints": "auto",
                    "spanNulls": false,
                    "stacking": {
                      "group": "A",
                      "mode": "none"
                    },
                    "thresholdsStyle": {
                      "mode": "off"
                    }
                  },
                  "mappings": [],
                  "thresholds": {
                    "mode": "absolute",
                    "steps": [
                      {
                        "color": "green",
                        "value": null
                      },
                      {
                        "color": "red",
                        "value": 80
                      }
                    ]
                  },
                  "unit": "percent"
                },
                "overrides": []
              },
              "gridPos": {
                "h": 8,
                "w": 12,
                "x": 12,
                "y": 84
              },
              "id": 9,
              "options": {
                "legend": {
                  "calcs": [],
                  "displayMode": "list",
                  "placement": "bottom",
                  "showLegend": true
                },
                "tooltip": {
                  "mode": "multi",
                  "sort": "desc"
                }
              },
              "targets": [
                {
                  "datasource": {
                    "type": "prometheus",
                    "uid": "PA58DA793C7250F1B"
                  },
                  "editorMode": "builder",
                  "expr": "sum by(pod) (rate(process_cpu_seconds_total{container=\"backend1\"}[$__rate_interval])) * 100 / 1",
                  "legendFormat": "__auto",
                  "range": true,
                  "refId": "A"
                }
              ],
              "title": "CPU",
              "type": "timeseries"
            }
          ],
          "title": "Backend1 (Prometheus metrics)",
          "type": "row"
        },
        {
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 27
          },
          "id": 23,
          "panels": [],
          "title": "Backend2",
          "type": "row"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 28
          },
          "id": 32,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "single",
              "sort": "none"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(http_status_code) (rate(http_server_duration_count{job=\"backend2-deployment\"}[5m]))",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "HTTP Server Responses",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              },
              "unit": "ms"
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 28
          },
          "id": 29,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.5, sum by(job, le) (rate(http_server_duration_bucket{job=\"backend2-deployment\"}[5m])))",
              "legendFormat": "50th percentile",
              "range": true,
              "refId": "A"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.95, sum by(job, le) (rate(http_server_duration_bucket{job=\"backend2-deployment\"}[5m])))",
              "hide": false,
              "legendFormat": "95th percentile",
              "range": true,
              "refId": "B"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.99, sum by(job, le) (rate(http_server_duration_bucket{job=\"backend2-deployment\"}[5m])))",
              "hide": false,
              "legendFormat": "99th percentile",
              "range": true,
              "refId": "C"
            }
          ],
          "title": "Server Request Duration",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              },
              "unit": "percentunit"
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 36
          },
          "id": 35,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": false
            },
            "tooltip": {
              "mode": "single",
              "sort": "none"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "process_runtime_jvm_cpu_utilization",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "CPU",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "description": "",
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 36
          },
          "id": 37,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(pool, type) (process_runtime_jvm_memory_usage)",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "Memory Usage",
          "type": "timeseries"
        }
      ],
      "schemaVersion": 37,
      "style": "dark",
      "tags": [],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-1h",
        "to": "now"
      },
      "timepicker": {},
      "timezone": "",
      "title": "Apps",
      "uid": "WbvDPqY4k",
      "version": 1,
      "weekStart": ""
    }
  collector.json: |
    {
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": {
              "type": "grafana",
              "uid": "-- Grafana --"
            },
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "target": {
              "limit": 100,
              "matchAny": false,
              "tags": [],
              "type": "dashboard"
            },
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "fiscalYearStartMonth": 0,
      "graphTooltip": 1,
      "links": [],
      "liveNow": false,
      "panels": [
        {
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 0
          },
          "id": 15,
          "panels": [],
          "title": "Receivers",
          "type": "row"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 9,
            "w": 12,
            "x": 0,
            "y": 1
          },
          "id": 2,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(receiver) (rate(otelcol_receiver_accepted_metric_points[$__rate_interval]))",
              "legendFormat": "{{receiver}} metrics",
              "range": true,
              "refId": "A"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(receiver) (rate(otelcol_receiver_accepted_spans[$__rate_interval]))",
              "hide": false,
              "legendFormat": "{{receiver}} traces",
              "range": true,
              "refId": "C"
            }
          ],
          "title": "Receiver Success",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 9,
            "w": 12,
            "x": 12,
            "y": 1
          },
          "id": 7,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(receiver) (rate(otelcol_receiver_refused_spans[$__rate_interval]))",
              "hide": false,
              "legendFormat": "{{receiver}} traces",
              "range": true,
              "refId": "B"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(receiver) (rate(otelcol_receiver_refused_metric_points[$__rate_interval]))",
              "hide": false,
              "legendFormat": "{{receiver}} metrics",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "Receiver Refused",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 10
          },
          "id": 31,
          "options": {
            "legend": {
              "calcs": [
                "lastNotNull"
              ],
              "displayMode": "table",
              "placement": "bottom",
              "showLegend": true,
              "sortBy": "Last *",
              "sortDesc": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(job) (scrape_samples_scraped)",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "Scrapes",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              },
              "unit": "s"
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 10
          },
          "id": 9,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "avg by(job) (scrape_duration_seconds)",
              "legendFormat": "{{job}}",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "Scrape Duration",
          "type": "timeseries"
        },
        {
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 18
          },
          "id": 21,
          "panels": [],
          "title": "Processors",
          "type": "row"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 0,
            "y": 19
          },
          "id": 23,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(processor) (rate(otelcol_processor_accepted_metric_points[$__rate_interval]))",
              "legendFormat": "{{processor}} metrics",
              "range": true,
              "refId": "A"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(processor) (rate(otelcol_processor_accepted_spans[$__rate_interval]))",
              "hide": false,
              "legendFormat": "{{processor}} spans",
              "range": true,
              "refId": "B"
            }
          ],
          "title": "Processor Accepted",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 8,
            "y": 19
          },
          "id": 25,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(processor) (rate(otelcol_processor_dropped_metric_points[$__rate_interval]))",
              "legendFormat": "{{processor}} metrics",
              "range": true,
              "refId": "A"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(processor) (rate(otelcol_processor_dropped_spans[$__rate_interval]))",
              "hide": false,
              "legendFormat": "{{processor}} traces",
              "range": true,
              "refId": "B"
            }
          ],
          "title": "Processor Dropped",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 16,
            "y": 19
          },
          "id": 27,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(processor) (rate(otelcol_processor_refused_metric_points[$__rate_interval]))",
              "legendFormat": "{{processor}} metrics",
              "range": true,
              "refId": "A"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(processor) (rate(otelcol_processor_refused_spans[$__rate_interval]))",
              "hide": false,
              "legendFormat": "{{processor}} traces",
              "range": true,
              "refId": "B"
            }
          ],
          "title": "Processor Refused",
          "type": "timeseries"
        },
        {
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 27
          },
          "id": 13,
          "panels": [],
          "title": "Exporters",
          "type": "row"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 0,
            "y": 28
          },
          "id": 17,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(exporter) (rate(otelcol_exporter_sent_metric_points[$__rate_interval]))",
              "legendFormat": "metrics {{exporter}}",
              "range": true,
              "refId": "A"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(exporter) (rate(otelcol_exporter_sent_spans[$__rate_interval]))",
              "hide": false,
              "legendFormat": "traces {{exporter}}",
              "range": true,
              "refId": "B"
            }
          ],
          "title": "Exporter Success",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 8,
            "y": 28
          },
          "id": 29,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(exporter) (rate(otelcol_exporter_enqueue_failed_log_records[$__rate_interval]))",
              "legendFormat": "{{exporter}} logs",
              "range": true,
              "refId": "A"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(exporter) (rate(otelcol_exporter_enqueue_failed_metric_points[$__rate_interval]))",
              "hide": false,
              "legendFormat": "{{exporter}} metrics",
              "range": true,
              "refId": "B"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(exporter) (rate(otelcol_exporter_enqueue_failed_spans[$__rate_interval]))",
              "hide": false,
              "legendFormat": "{{exporter}} traces",
              "range": true,
              "refId": "C"
            }
          ],
          "title": "Exporter Enqueue Failures",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 16,
            "y": 28
          },
          "id": 4,
          "options": {
            "legend": {
              "calcs": [
                "min",
                "mean",
                "max",
                "last"
              ],
              "displayMode": "table",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(exporter) (rate(otelcol_exporter_send_failed_metric_points[$__rate_interval]))",
              "hide": false,
              "legendFormat": "{{exporter}} metrics",
              "range": true,
              "refId": "B"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(exporter) (rate(otelcol_exporter_send_failed_spans[$__rate_interval]))",
              "hide": false,
              "legendFormat": "{{exporter}} traces",
              "range": true,
              "refId": "C"
            }
          ],
          "title": "exporter failures",
          "type": "timeseries"
        },
        {
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 36
          },
          "id": 11,
          "panels": [],
          "title": "Spanmetrics",
          "type": "row"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 37
          },
          "id": 6,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(service_name, span_name) (rate(converted_calls{span_kind=\"SPAN_KIND_SERVER\"}[$__rate_interval]))",
              "legendFormat": "{{service_name}} - {{span_name}}",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "Server Calls",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              },
              "unit": "ms"
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 37
          },
          "id": 19,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.5, sum by(le) (rate(converted_duration_bucket{span_kind=\"SPAN_KIND_SERVER\"}[$__rate_interval])))",
              "legendFormat": "50th percentile",
              "range": true,
              "refId": "A"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.95, sum by(le) (rate(converted_duration_bucket{span_kind=\"SPAN_KIND_SERVER\"}[$__rate_interval])))",
              "hide": false,
              "legendFormat": "95th percentile",
              "range": true,
              "refId": "B"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.99, sum by(le) (rate(converted_duration_bucket{span_kind=\"SPAN_KIND_SERVER\"}[$__rate_interval])))",
              "hide": false,
              "legendFormat": "99th percentile",
              "range": true,
              "refId": "C"
            }
          ],
          "title": "Server Calls Latency",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 45
          },
          "id": 32,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(service_name, span_name) (rate(converted_calls{span_kind=\"SPAN_KIND_CLIENT\"}[$__rate_interval]))",
              "legendFormat": "{{service_name}} - {{span_name}}",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "Client Calls",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 45
          },
          "id": 33,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(service_name, span_name) (rate(converted_calls{span_kind=\"SPAN_KIND_INTERNAL\"}[$__rate_interval]))",
              "legendFormat": "{{service_name}} - {{span_name}}",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "Internal Calls",
          "type": "timeseries"
        }
      ],
      "schemaVersion": 37,
      "style": "dark",
      "tags": [],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-1h",
        "to": "now"
      },
      "timepicker": {},
      "timezone": "",
      "title": "Collector",
      "uid": "7hHiATL4z",
      "version": 1,
      "weekStart": ""
    }
  target-allocator.json: |
    {
      "annotations": {
        "list": [
          {
            "builtIn": 1,
            "datasource": {
              "type": "grafana",
              "uid": "-- Grafana --"
            },
            "enable": true,
            "hide": true,
            "iconColor": "rgba(0, 211, 255, 1)",
            "name": "Annotations & Alerts",
            "target": {
              "limit": 100,
              "matchAny": false,
              "tags": [],
              "type": "dashboard"
            },
            "type": "dashboard"
          }
        ]
      },
      "editable": true,
      "fiscalYearStartMonth": 0,
      "graphTooltip": 1,
      "id": 5,
      "links": [],
      "liveNow": false,
      "panels": [
        {
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 0
          },
          "id": 22,
          "title": "Service Discovery",
          "type": "row"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 0,
            "y": 1
          },
          "id": 24,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "single",
              "sort": "none"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(event) (rate(prometheus_sd_kubernetes_events_total[$__rate_interval]))",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "SD Events",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 12,
            "x": 12,
            "y": 1
          },
          "id": 26,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(pod) (rate(prometheus_sd_http_failures_total[$__rate_interval]))",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "SD HTTP Failures",
          "type": "timeseries"
        },
        {
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 9
          },
          "id": 18,
          "title": "Allocation",
          "type": "row"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 0,
            "y": 10
          },
          "id": 8,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "avg by(collector_name) (opentelemetry_allocator_targets_per_collector)",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "Targets Per Collector",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 8,
            "y": 10
          },
          "id": 6,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "avg(opentelemetry_allocator_collectors_allocatable)",
              "legendFormat": "allocatable",
              "range": true,
              "refId": "A"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "avg(opentelemetry_allocator_collectors_discovered)",
              "hide": false,
              "legendFormat": "discovered",
              "range": true,
              "refId": "B"
            }
          ],
          "title": "Num Collectors",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              },
              "unit": "s"
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 16,
            "y": 10
          },
          "id": 12,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.5, sum by(le) (rate(opentelemetry_allocator_time_to_allocate_bucket[$__rate_interval])))",
              "legendFormat": "50th percentile",
              "range": true,
              "refId": "A"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.95, sum by(le) (rate(opentelemetry_allocator_time_to_allocate_bucket[$__rate_interval])))",
              "hide": false,
              "legendFormat": "95th percentile",
              "range": true,
              "refId": "B"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.99, sum by(le) (rate(opentelemetry_allocator_time_to_allocate_bucket[$__rate_interval])))",
              "hide": false,
              "legendFormat": "99th percentile",
              "range": true,
              "refId": "C"
            }
          ],
          "title": "Allocation Duration",
          "type": "timeseries"
        },
        {
          "collapsed": false,
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 18
          },
          "id": 20,
          "panels": [],
          "title": "Server",
          "type": "row"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 0,
            "y": 19
          },
          "id": 30,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(path) (rate(opentelemetry_allocator_http_duration_seconds_sum[$__rate_interval]))",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "Requests",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              },
              "unit": "s"
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 8,
            "y": 19
          },
          "id": 10,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.5, sum by(le) (rate(opentelemetry_allocator_http_duration_seconds_bucket[$__rate_interval])))",
              "legendFormat": "50th percentile",
              "range": true,
              "refId": "A"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.95, sum by(le) (rate(opentelemetry_allocator_http_duration_seconds_bucket[$__rate_interval])))",
              "hide": false,
              "legendFormat": "95th percentile",
              "range": true,
              "refId": "B"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "histogram_quantile(0.99, sum by(le) (rate(opentelemetry_allocator_http_duration_seconds_bucket[$__rate_interval])))",
              "hide": false,
              "legendFormat": "99th percentile",
              "range": true,
              "refId": "C"
            }
          ],
          "title": "HTTP Request Duration",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 16,
            "y": 19
          },
          "id": 28,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(code, pod) (rate(promhttp_metric_handler_requests_total[$__rate_interval]))",
              "legendFormat": "{{pod}} - {{code}}",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "Metrics Handler Requests",
          "type": "timeseries"
        },
        {
          "gridPos": {
            "h": 1,
            "w": 24,
            "x": 0,
            "y": 27
          },
          "id": 16,
          "title": "Machine",
          "type": "row"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "description": "This assumes one core - to adjust, modify the \"Divide by scalar\" value to the number of cores you're running",
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              },
              "unit": "percentunit"
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 0,
            "y": 28
          },
          "id": 14,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(pod) (rate(process_cpu_seconds_total{service=\"otel-prom-cr-targetallocator\"}[$__rate_interval])) * 100 / 1",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "CPU",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              },
              "unit": "percentunit"
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 8,
            "y": 28
          },
          "id": 4,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "multi",
              "sort": "desc"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(pod) (go_memstats_alloc_bytes{service=\"otel-prom-cr-targetallocator\"})",
              "hide": true,
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            },
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(pod) (go_memstats_sys_bytes{service=\"otel-prom-cr-targetallocator\"})",
              "hide": true,
              "legendFormat": "max {{pod}}",
              "range": true,
              "refId": "B"
            },
            {
              "datasource": {
                "name": "Expression",
                "type": "__expr__",
                "uid": "__expr__"
              },
              "expression": "$A/$B",
              "hide": false,
              "refId": "pod",
              "type": "math"
            }
          ],
          "title": "Memory",
          "type": "timeseries"
        },
        {
          "datasource": {
            "type": "prometheus",
            "uid": "PA58DA793C7250F1B"
          },
          "fieldConfig": {
            "defaults": {
              "color": {
                "mode": "palette-classic"
              },
              "custom": {
                "axisCenteredZero": false,
                "axisColorMode": "text",
                "axisLabel": "",
                "axisPlacement": "auto",
                "barAlignment": 0,
                "drawStyle": "line",
                "fillOpacity": 0,
                "gradientMode": "none",
                "hideFrom": {
                  "legend": false,
                  "tooltip": false,
                  "viz": false
                },
                "lineInterpolation": "linear",
                "lineWidth": 1,
                "pointSize": 5,
                "scaleDistribution": {
                  "type": "linear"
                },
                "showPoints": "auto",
                "spanNulls": false,
                "stacking": {
                  "group": "A",
                  "mode": "none"
                },
                "thresholdsStyle": {
                  "mode": "off"
                }
              },
              "mappings": [],
              "thresholds": {
                "mode": "absolute",
                "steps": [
                  {
                    "color": "green",
                    "value": null
                  },
                  {
                    "color": "red",
                    "value": 80
                  }
                ]
              }
            },
            "overrides": []
          },
          "gridPos": {
            "h": 8,
            "w": 8,
            "x": 16,
            "y": 28
          },
          "id": 2,
          "options": {
            "legend": {
              "calcs": [],
              "displayMode": "list",
              "placement": "bottom",
              "showLegend": true
            },
            "tooltip": {
              "mode": "single",
              "sort": "none"
            }
          },
          "targets": [
            {
              "datasource": {
                "type": "prometheus",
                "uid": "PA58DA793C7250F1B"
              },
              "editorMode": "builder",
              "expr": "sum by(pod) (go_goroutines{service=\"otel-prom-cr-targetallocator\"})",
              "legendFormat": "__auto",
              "range": true,
              "refId": "A"
            }
          ],
          "title": "Goroutines",
          "type": "timeseries"
        }
      ],
      "schemaVersion": 37,
      "style": "dark",
      "tags": [],
      "templating": {
        "list": []
      },
      "time": {
        "from": "now-1h",
        "to": "now"
      },
      "timepicker": {},
      "timezone": "",
      "title": "Target Allocator",
      "uid": "ulLjw3L4z",
      "version": 4,
      "weekStart": ""
    }
  logs-otel.json: |
    {
    "annotations": {
      "list": [
        {
          "builtIn": 1,
          "datasource": {
            "type": "grafana",
            "uid": "-- Grafana --"
          },
          "enable": true,
          "hide": true,
          "iconColor": "rgba(0, 211, 255, 1)",
          "name": "Annotations & Alerts",
          "target": {
            "limit": 100,
            "matchAny": false,
            "tags": [],
            "type": "dashboard"
          },
          "type": "dashboard"
        }
      ]
    },
    "editable": true,
    "fiscalYearStartMonth": 0,
    "graphTooltip": 0,
    "id": 3,
    "links": [],
    "liveNow": false,
    "panels": [
      {
        "datasource": {
          "type": "loki",
          "uid": "PEA2100DC89AE9FE2"
        },
        "gridPos": {
          "h": 9,
          "w": 12,
          "x": 0,
          "y": 0
        },
        "id": 2,
        "options": {
          "dedupStrategy": "none",
          "enableLogDetails": true,
          "prettifyLogMessage": false,
          "showCommonLabels": false,
          "showLabels": false,
          "showTime": false,
          "sortOrder": "Descending",
          "wrapLogMessage": false
        },
        "pluginVersion": "9.1.0",
        "targets": [
          {
            "datasource": {
            "type": "loki",
            "uid": "PEA2100DC89AE9FE2"
            },
            "editorMode": "builder",
            "expr": "{exporter=\"OTLP\"} |= ``",
            "queryType": "range",
            "refId": "A"
          }
        ],
        "title": "OpenTelemetry Logs",
        "type": "logs"
      }
    ],
    "schemaVersion": 37,
    "style": "dark",
    "tags": [],
    "templating": {
      "list": []
    },
    "time": {
      "from": "now-5m",
      "to": "now"
    },
    "timepicker": {},
    "timezone": "",
    "title": "Loki Dashboard",
    "uid": "WfV_7jY4k",
    "version": 2,
    "weekStart": ""
    }
kind: ConfigMap
metadata:
  name: grafana-dashboards-demo
  namespace: observability-backend
---
apiVersion: v1
data:
  datasources.yml: |
    apiVersion: 1
    datasources:
        - access: proxy
          basicAuth: false
          editable: false
          isDefault: false
          jsonData:
            derivedFields:
                - datasourceUid: tempo
                  matcherRegex: (?:traceID|trace_id)=(\w+)
                  name: TraceID
                  url: $${__value.raw}
            maxLines: 1000
          name: Logs
          type: loki
          url: http://loki.observability-backend.svc.cluster.local:3100
          version: 1
        - access: proxy
          basicAuth: false
          editable: false
          isDefault: true
          jsonData:
            disableMetricsLookup: false
            exemplarTraceIdDestinations:
                - datasourceUid: tempo
                  name: traceID
            httpMethod: POST
          name: Metrics
          type: prometheus
          url: http://mimir.observability-backend.svc.cluster.local/prometheus
          version: 1
        - access: browser
          basicAuth: false
          editable: false
          isDefault: false
          jsonData:
            tracesToLogs:
                datasourceUid: Loki
                tags:
                    - job
                    - instance
                    - pod
                    - namespace
          name: Traces
          type: tempo
          uid: tempo
          url: http://tempo.observability-backend.svc.cluster.local/
          version: 1
kind: ConfigMap
metadata:
  labels: {}
  name: grafana-datasources
  namespace: observability-backend
---
apiVersion: v1
kind: ConfigMap
metadata:
  labels: {}
  name: grafana-notification-channels
  namespace: observability-backend
---
apiVersion: v1
kind: Service
metadata:
  labels:
    name: grafana
  name: grafana
  namespace: observability-backend
spec:
  ports:
  - name: grafana-grafana-metrics
    port: 3000
    targetPort: 3000
  - name: http
    port: 80
    targetPort: 3000
  selector:
    name: grafana
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: observability-backend
spec:
  minReadySeconds: 10
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      name: grafana
  template:
    metadata:
      labels:
        name: grafana
    spec:
      containers:
      - env:
        - name: GF_INSTALL_PLUGINS
        - name: GF_PATHS_CONFIG
          value: /etc/grafana-config/grafana.ini
        image: grafana/grafana:9.2.3
        imagePullPolicy: IfNotPresent
        name: grafana
        ports:
        - containerPort: 3000
          name: grafana-metrics
        resources:
          requests:
            cpu: 10m
            memory: 40Mi
        volumeMounts:
        - mountPath: /etc/grafana-config
          name: grafana-config
        - mountPath: /etc/grafana/provisioning/dashboards
          name: grafana-dashboard-provisioning
        - mountPath: /grafana/dashboards
          name: grafana-dashboards
        - mountPath: /grafana/dashboards-demo
          name: grafana-dashboards-demo
        - mountPath: /etc/grafana/provisioning/datasources
          name: grafana-datasources
        - mountPath: /etc/grafana/provisioning/notifiers
          name: grafana-notification-channels
      volumes:
      - configMap:
          name: grafana-config
        name: grafana-config
      - configMap:
          name: grafana-dashboard-provisioning
        name: grafana-dashboard-provisioning
      - configMap:
          name: grafana-dashboards
        name: grafana-dashboards
      - configMap:
          name: grafana-dashboards-demo
        name: grafana-dashboards-demo
      - configMap:
          name: grafana-datasources
        name: grafana-datasources
      - configMap:
          name: grafana-notification-channels
        name: grafana-notification-channels
        ```

```bash
kubectl apply -f backend/backend-01.yaml
```

![alt text](deploy-backend01.png)

**Check namespaces using kubens:**

```bash
kubens
```

![alt text](kubens1.png)

Your output should look like above.

My backend is now located in the observability-backend namespace and with **port-forwarding** the grafana service i can access my grafana dashboard which is available for visualisation with preconfigured datasources.

```bash
kubectl port-forward -n observability-backend svc/grafana 3000:3000
```

![alt text](port-forward-grafana2-3000-1.png)

---

### Implementing OpenTelemetry Collector

Well,  what is the OpenTelemetry Collector and why do i need it? 

> The OpenTelemetry Collector is a powerful, flexible, vendor-agnostic tool for collecting, processing, and exporting telemetry data. Its modular architecture comprises several key components working together to form a robust data pipeline.

**Here's a breakdown of the OpenTelemetry Collector's core components:**

* **Receivers:**  The entry point for your telemetry data. They gather data from various sources, such as:
    - **Applications:**  Instrumented with OpenTelemetry SDKs.
    - **Infrastructure:** Metrics and logs from systems like Kubernetes or Prometheus.
    - **Protocols:** Data received via protocols like OTLP (OpenTelemetry Protocol), Zipkin, or Jaeger.

    Receivers convert the collected data into a standardized format called **pData** (pipeline data) for use within the Collector.

* **Processors:** Act as the transformation layer. They manipulate the pData received from Receivers, allowing you to:
    - **Filter data:** Include or exclude specific data points based on defined criteria.
    - **Enrich data:** Add valuable context or metadata to improve analysis.
    - **Transform data:**  Modify data formats or values to meet specific requirements. 

    Examples include batch processors for efficient data handling and renaming processors for consistent naming conventions. 

* **Exporters:** Responsible for sending processed telemetry data to backend systems for storage, visualization, and analysis.  Common destinations include:
    - **OpenTelemetry Protocol (OTLP):** A vendor-neutral standard for telemetry data exchange.
    - **Monitoring Systems:**  Prometheus, Jaeger, Zipkin, and other popular tools.
    - **Logging Backends:** Elasticsearch, Splunk, etc., for log aggregation and analysis.

* **Extensions:** Extend the Collector's core functionality by:
    - **Adding authentication/authorization:**  Securely connect to external services.
    - **Enabling remote sampling:** Control data ingestion rates dynamically.
    - **Integrating with other systems:** Enhance interoperability with your existing observability stack.

* **Connectors:**  Bridge the gap between different pipelines, acting as both an Exporter and a Receiver.  They allow you to:
    - **Route data between pipelines:**  Create complex data flow scenarios.
    - **Process data in stages:** Apply different transformations based on the data's destination.

**To learn more about the OpenTelemetry Collector Distributions ([core](https://github.com/open-telemetry/opentelemetry-collector-releases/blob/v0.74.0/distributions/otelcol/manifest.yaml) and [contrib](https://github.com/open-telemetry/opentelemetry-collector-releases/blob/v0.74.0/distributions/otelcol-contrib/manifest.yaml)), [OpenTelemetry Collector Builder](https://github.com/open-telemetry/opentelemetry-collector/blob/v0.74.0/cmd/builder) and Configuration options of individual components, explore the official OpenTelemetry Collector documentation:** [https://opentelemetry.io/docs/collector/](https://opentelemetry.io/docs/collector/)

Ok! Enough theory. back to practicals!



