# observability

Metrics, logs and dashboards for the store platform.

| Component  | Role                                        | Access                                   |
|------------|---------------------------------------------|------------------------------------------|
| Prometheus | Scrapes metrics from labeled containers     | `http://<gateway>/prometheus/` (store login, administrator role) |
| Loki       | Stores logs                                 | internal (`loki:3100`)                   |
| Alloy      | Reads container stdout and ships it to Loki | internal                                 |
| Grafana    | Dashboards (`Store / Store - Services`)     | `http://<gateway>/grafana/` (public route, Grafana login) |

Nothing is published on host ports: like every other service, Grafana and Prometheus
are reached through the gateway, which routes `/grafana/**` and `/prometheus/**`.
Both are served under those sub-paths (`GF_SERVER_SERVE_FROM_SUB_PATH`, `--web.route-prefix`).

## How it is used

The platform includes this repo as a submodule and pulls it into its compose
file, so everything runs on the same network:

```yaml
# platform/api/compose.yaml
include:
  - ./observability/compose.yaml
```

## Connecting a service

Nothing in this repo changes when a service is added. Each service opts in
with labels on its compose entry:

```yaml
order:
  labels:
    prometheus.scrape: "true"
    prometheus.port: "8080"
    prometheus.path: /order/actuator/prometheus
    logs.collect: "true"
```

For a Spring Boot service, the metrics endpoint needs:

- `spring-boot-starter-actuator` and `micrometer-registry-prometheus`
- `management.endpoints.web.exposure.include: prometheus`
- `management.metrics.distribution.percentiles-histogram.http.server.requests: true`
  (lets the dashboard compute p95 latency)
- `logging.structured.format.console: ecs` (JSON logs, so Loki gets a `level` label)

## Configuration

| Variable                 | Default |
|--------------------------|---------|
| `GRAFANA_ADMIN_USER`     | `admin` |
| `GRAFANA_ADMIN_PASSWORD` | `admin` |
| `PROMETHEUS_RETENTION`   | `7d`    |
| `OBSERVABILITY_PUBLIC_URL` | `http://localhost:8080` (the gateway's public address, used in links and redirects) |

Prometheus and Alloy read `/var/run/docker.sock` (read-only) to discover containers.

## Kubernetes

`kustomization.yaml` (repository root) renders the same stack for the `store`
namespace, reusing `loki/loki.yaml`, the Grafana provisioning and the dashboard.
Only discovery changes, since there is no Docker socket in a cluster:

| | docker compose | Kubernetes |
|---|---|---|
| Prometheus targets | containers labeled `prometheus.scrape=true` | pods annotated `prometheus.io/scrape: "true"` (`k8s/prometheus.yml`) |
| Logs | containers labeled `logs.collect=true` | every pod of the `store` namespace (`k8s/config.alloy`) |
| Permissions | read-only Docker socket | `Role` limited to reading pods (and pod logs, for Alloy) |

A service opts in to metrics with annotations on its pod template:

```yaml
template:
  metadata:
    annotations:
      prometheus.io/scrape: "true"
      prometheus.io/port: "8080"
      prometheus.io/path: /order/actuator/prometheus
```

The platform consumes it in `platform/k8s/base/kustomization.yaml`. Grafana
reads `GF_SECURITY_ADMIN_USER` and `GF_SECURITY_ADMIN_PASSWORD` from the
`grafana-secret` Secret, and `OBSERVABILITY_PUBLIC_URL` comes from the
`observability-config` ConfigMap.
