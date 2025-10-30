# Grafana Alloy Examples

Grafana Alloy is the new, unified agent for collecting telemetry data (logs, metrics, traces). It's more flexible and powerful than Promtail.

## Quick Start with Docker

```bash
docker run -d \
  --name alloy \
  -v $(pwd)/config.alloy:/etc/alloy/config.alloy:ro \
  -v /var/log:/var/log:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  grafana/alloy:latest \
  run --server.http.listen-addr=0.0.0.0:12345 /etc/alloy/config.alloy
```

## Docker Compose

```yaml
version: '3.8'
services:
  alloy:
    image: grafana/alloy:latest
    volumes:
      - ./config.alloy:/etc/alloy/config.alloy:ro
      - /var/log:/var/log:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command:
      - run
      - --server.http.listen-addr=0.0.0.0:12345
      - /etc/alloy/config.alloy
    ports:
      - "12345:12345"
    restart: unless-stopped
```

## Configuration

Edit `config.alloy` and update the Loki endpoint:

```alloy
loki.write "default" {
  endpoint {
    url = "https://your-app-name.ondigitalocean.app/loki/api/v1/push"
  }
}
```

Replace `your-app-name` with your actual App Platform app name.

## Key Features

### 1. Multiple Data Types

Alloy can collect:
- **Logs** via `loki.source.*`
- **Metrics** via `prometheus.scrape`
- **Traces** via `otelcol.receiver.*`

### 2. Pipeline Processing

```alloy
loki.source.file "logs" {
  targets = [...]

  // Parse JSON
  stage.json {
    expressions = {
      level = "level",
    }
  }

  // Filter by level
  stage.match {
    selector = "{level=\"error\"}"
    action   = "keep"
  }

  forward_to = [loki.write.default.receiver]
}
```

### 3. Service Discovery

Automatically discover targets:

```alloy
// Docker containers
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

loki.source.docker "containers" {
  targets    = discovery.docker.containers.targets
  forward_to = [loki.write.default.receiver]
}

// Kubernetes pods
discovery.kubernetes "pods" {
  role = "pod"
}

loki.source.kubernetes "pods" {
  targets    = discovery.kubernetes.pods.targets
  forward_to = [loki.write.default.receiver]
}
```

## Common Patterns

### Application Logs with JSON Parsing

```alloy
loki.source.file "app" {
  targets = [
    {
      __path__ = "/app/logs/*.json",
      job      = "myapp",
    },
  ]

  stage.json {
    expressions = {
      timestamp = "timestamp",
      level     = "level",
      message   = "message",
      caller    = "caller",
    }
  }

  stage.labels {
    values = {
      level = "",
    }
  }

  stage.timestamp {
    source = "timestamp"
    format = "RFC3339"
  }

  forward_to = [loki.write.default.receiver]
}
```

### Multi-Line Logs (Stack Traces)

```alloy
loki.source.file "multiline" {
  targets = [...]

  stage.multiline {
    firstline     = "^\\d{4}-\\d{2}-\\d{2}"
    max_wait_time = "3s"
  }

  forward_to = [loki.write.default.receiver]
}
```

### Sampling High-Volume Logs

```alloy
loki.source.file "high_volume" {
  targets = [...]

  stage.sampling {
    rate = 0.1  // Keep 10%
  }

  forward_to = [loki.write.default.receiver]
}
```

### Filtering Logs

```alloy
loki.source.file "filtered" {
  targets = [...]

  // Drop debug logs
  stage.match {
    selector = "{job=\"myapp\"}"

    stage.drop {
      expression = ".*debug.*"
    }
  }

  // Only keep errors and warnings
  stage.match {
    selector = "{level!~\"error|warning\"}"
    action   = "drop"
  }

  forward_to = [loki.write.default.receiver]
}
```

## Testing Configuration

1. **Validate syntax**:
   ```bash
   docker run --rm \
     -v $(pwd)/config.alloy:/etc/alloy/config.alloy:ro \
     grafana/alloy:latest \
     fmt /etc/alloy/config.alloy
   ```

2. **Run in debug mode**:
   ```bash
   docker run --rm \
     -v $(pwd)/config.alloy:/etc/alloy/config.alloy:ro \
     grafana/alloy:latest \
     run --server.http.listen-addr=0.0.0.0:12345 --log.level=debug /etc/alloy/config.alloy
   ```

3. **Check UI**:
   Open http://localhost:12345 to see Alloy's built-in UI with component graph and metrics.

## Monitoring Alloy

Alloy exposes metrics about itself:

```alloy
prometheus.exporter.self "default" { }

prometheus.scrape "default" {
  targets    = prometheus.exporter.self.default.targets
  forward_to = [prometheus.remote_write.default.receiver]
}
```

Key metrics:
- `alloy_component_evaluation_duration_seconds`
- `alloy_loki_write_entries_sent_total`
- `alloy_loki_write_sent_bytes_total`

## Troubleshooting

### Configuration Errors

```bash
# Validate config
docker run --rm \
  -v $(pwd)/config.alloy:/config.alloy:ro \
  grafana/alloy:latest \
  run --server.http.listen-addr=0.0.0.0:12345 --config.file=/config.alloy
```

### No Logs Being Sent

1. Check Alloy UI: http://localhost:12345
2. Verify file paths exist and are readable
3. Check Loki connectivity:
   ```bash
   curl https://your-app-name.ondigitalocean.app/loki/ready
   ```

### High Memory Usage

- Reduce batch sizes
- Increase forward frequency
- Sample high-volume logs

## Alloy vs Promtail

| Feature | Alloy | Promtail |
|---------|-------|----------|
| Logs | ✅ | ✅ |
| Metrics | ✅ | ❌ |
| Traces | ✅ | ❌ |
| UI | ✅ Built-in | ❌ |
| Configuration | River (HCL-like) | YAML |
| Flexibility | High | Medium |
| Learning Curve | Steeper | Gentler |

**When to use Alloy**:
- Need unified agent for logs + metrics + traces
- Want powerful pipeline processing
- Need service discovery
- Want built-in UI for debugging

**When to use Promtail**:
- Only need log collection
- Prefer simpler YAML config
- Already familiar with Promtail

## Additional Resources

- **Alloy Documentation**: https://grafana.com/docs/alloy/latest/
- **Configuration Reference**: https://grafana.com/docs/alloy/latest/reference/
- **Examples**: https://github.com/grafana/alloy/tree/main/example
- **Migration from Promtail**: https://grafana.com/docs/alloy/latest/tasks/migrate/from-promtail/
