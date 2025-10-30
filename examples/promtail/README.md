# Promtail Examples

Promtail is the official log agent for Loki. It reads logs from files and forwards them to Loki.

## Quick Start with Docker

```bash
docker run -d \
  --name promtail \
  -v $(pwd)/promtail-config.yaml:/etc/promtail/config.yaml \
  -v /var/log:/var/log:ro \
  grafana/promtail:latest \
  -config.file=/etc/promtail/config.yaml
```

## Docker Compose

```yaml
version: '3'
services:
  promtail:
    image: grafana/promtail:latest
    volumes:
      - ./promtail-config.yaml:/etc/promtail/config.yaml
      - /var/log:/var/log:ro
    command: -config.file=/etc/promtail/config.yaml
    restart: unless-stopped
```

## Configuration

Edit `promtail-config.yaml` and update:

```yaml
clients:
  - url: https://your-app-name.ondigitalocean.app/loki/api/v1/push
```

Replace `your-app-name` with your actual App Platform app name.

### For Internal Access

If running Promtail as a service in the same App Platform app:

```yaml
clients:
  - url: http://loki:3100/loki/api/v1/push
```

## Testing

1. **Start Promtail**:
   ```bash
   docker-compose up -d
   ```

2. **Check logs**:
   ```bash
   docker logs promtail
   ```

3. **Generate test logs**:
   ```bash
   echo "Test log message" >> /var/log/test.log
   ```

4. **Query in Grafana**:
   - Go to Explore in Grafana
   - Select Loki datasource
   - Query: `{job="varlogs"}`

## Common Configurations

### Application Logs (JSON Format)

```yaml
scrape_configs:
  - job_name: app
    static_configs:
      - targets:
          - localhost
        labels:
          job: myapp
          environment: production
          __path__: /app/logs/*.json
    pipeline_stages:
      - json:
          expressions:
            level: level
            message: message
      - labels:
          level:
```

### Docker Container Logs

```yaml
scrape_configs:
  - job_name: docker
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          __path__: /var/lib/docker/containers/*/*.log
```

### Kubernetes Pods (if applicable)

```yaml
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
```

## Filtering Logs

### Exclude Debug Logs

```yaml
pipeline_stages:
  - match:
      selector: '{job="myapp"}'
      stages:
        - drop:
            expression: ".*debug.*"
```

### Sample High-Volume Logs

```yaml
pipeline_stages:
  - match:
      selector: '{job="high-volume"}'
      stages:
        - sampling:
            rate: 0.1  # Keep 10% of logs
```

## Troubleshooting

### Promtail Not Starting

Check logs:
```bash
docker logs promtail
```

Common issues:
- Config file syntax errors
- File permissions (ensure Promtail can read log files)
- Network connectivity to Loki

### No Logs Appearing in Loki

1. **Check Promtail metrics**:
   ```bash
   curl http://localhost:9080/metrics | grep promtail_sent_entries_total
   ```

2. **Verify log paths exist**:
   ```bash
   ls -la /var/log/*.log
   ```

3. **Test Loki connectivity**:
   ```bash
   curl https://your-app-name.ondigitalocean.app/loki/ready
   ```

### Position File Issues

If Promtail re-sends old logs:
```bash
# Remove position file to restart from current
rm /tmp/positions.yaml
docker restart promtail
```

## Performance Tuning

### Increase Batch Size

```yaml
clients:
  - url: http://loki:3100/loki/api/v1/push
    batchwait: 1s
    batchsize: 1048576  # 1MB
```

### Limit File Discovery

```yaml
scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          __path__: /var/log/syslog  # Specific file instead of wildcard
```

## Additional Resources

- **Promtail Documentation**: https://grafana.com/docs/loki/latest/clients/promtail/
- **Configuration Reference**: https://grafana.com/docs/loki/latest/clients/promtail/configuration/
- **Pipeline Stages**: https://grafana.com/docs/loki/latest/clients/promtail/stages/
