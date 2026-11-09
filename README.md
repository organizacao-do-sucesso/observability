# observability

Metrics, logs and dashboards for the store platform.

| Component  | Role                                             | URL                    |
|------------|--------------------------------------------------|------------------------|
| Prometheus | Scrapes metrics from labeled containers          | http://localhost:9090  |
| Loki       | Stores logs                                      | internal (`loki:3100`) |
| Alloy      | Reads container stdout and ships it to Loki      | internal               |
| Grafana    | Dashboards (`Store / Store - Services`)          | http://localhost:3000  |

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

Prometheus and Alloy read `/var/run/docker.sock` (read-only) to discover containers.
