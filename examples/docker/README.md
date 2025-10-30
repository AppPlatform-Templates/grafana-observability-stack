# Docker Loki Logging Driver

The Docker Loki logging driver allows containers to send logs directly to Loki without needing a separate log agent.

## Installation

### Install Loki Docker Plugin

```bash
docker plugin install grafana/loki-docker-driver:latest --alias loki --grant-all-permissions
```

### Verify Installation

```bash
docker plugin ls
```

You should see:
```
ID             NAME          DESCRIPTION           ENABLED
ac720b8fcfdb   loki:latest   Loki Logging Driver   true
```

## Usage

### Single Container

```bash
docker run \
  --log-driver=loki \
  --log-opt loki-url="https://your-app-name.ondigitalocean.app/loki/api/v1/push" \
  --log-opt loki-external-labels="job=test,environment=dev" \
  your-image:latest
```

### Docker Compose

See `docker-compose.yaml` for complete example.

```yaml
services:
  myapp:
    image: myapp:latest
    logging:
      driver: loki
      options:
        loki-url: "https://your-app-name.ondigitalocean.app/loki/api/v1/push"
        loki-external-labels: "job=myapp,environment=production"
```

## Configuration Options

| Option | Description | Default |
|--------|-------------|---------|
| `loki-url` | Loki push endpoint URL | Required |
| `loki-external-labels` | Additional labels (comma-separated) | `""` |
| `loki-timeout` | Timeout for sending logs | `10s` |
| `loki-batch-size` | Batch size before sending | `102400` (100KB) |
| `loki-batch-wait` | Max wait time before sending batch | `1s` |
| `loki-retries` | Number of retries | `10` |
| `loki-max-backoff` | Max backoff time between retries | `3s` |
| `loki-min-backoff` | Min backoff time between retries | `100ms` |
| `loki-format` | Log format (`json` or `raw`) | `raw` |

## Labels

### Static Labels

Add labels directly in the configuration:

```yaml
logging:
  options:
    loki-external-labels: "job=myapp,environment=prod,region=us-east"
```

### Dynamic Labels (Template Variables)

Use Docker template variables:

```yaml
logging:
  options:
    loki-external-labels: "job=myapp,container={{.Name}},image={{.ImageName}}"
```

**Available variables**:
- `{{.Name}}` - Container name
- `{{.FullID}}` - Full container ID
- `{{.ID}}` - Short container ID (12 chars)
- `{{.ImageID}}` - Image ID
- `{{.ImageFullID}}` - Full image ID
- `{{.ImageName}}` - Image name

### Relabeling in Loki

You can extract labels from log lines in Loki config:

```yaml
# In loki/config.yaml (not recommended for Docker driver)
limits_config:
  reject_old_samples: true
  reject_old_samples_max_age: 168h
```

## JSON Logs

If your application logs in JSON format:

```yaml
logging:
  driver: loki
  options:
    loki-url: "https://your-app-name.ondigitalocean.app/loki/api/v1/push"
    loki-format: "json"  # Preserve JSON structure
```

Then in Grafana, parse with LogQL:
```logql
{job="myapp"} | json | level="error"
```

## Setting as Default Driver

### Docker Daemon Configuration

Edit `/etc/docker/daemon.json`:

```json
{
  "log-driver": "loki",
  "log-opts": {
    "loki-url": "https://your-app-name.ondigitalocean.app/loki/api/v1/push",
    "loki-batch-size": "400",
    "loki-external-labels": "job=docker,host=myhost"
  }
}
```

Restart Docker:
```bash
sudo systemctl restart docker
```

Now all containers will use Loki by default (unless overridden).

## Performance Tuning

### For High-Volume Logs

```yaml
logging:
  options:
    loki-batch-size: "1048576"  # 1MB
    loki-batch-wait: "5s"
    loki-max-backoff: "5s"
```

### For Low-Latency

```yaml
logging:
  options:
    loki-batch-size: "102400"  # 100KB
    loki-batch-wait: "1s"
    loki-max-backoff: "1s"
```

## Troubleshooting

### Driver Not Found

```bash
# Check if plugin is installed
docker plugin ls

# If not, install it
docker plugin install grafana/loki-docker-driver:latest --alias loki --grant-all-permissions
```

### Logs Not Appearing in Loki

1. **Check Docker logs**:
   ```bash
   docker logs <container-id>
   ```
   Should show empty if Loki driver is working.

2. **Check plugin logs**:
   ```bash
   journalctl -u docker.service | grep loki
   ```

3. **Test Loki connectivity**:
   ```bash
   curl https://your-app-name.ondigitalocean.app/loki/ready
   ```

4. **Verify labels in Grafana**:
   - Go to Explore
   - Try: `{job="myapp"}`

### Connection Refused

If using internal URL (`http://loki:3100`), ensure:
- Docker container is in same network as Loki
- Or use external HTTPS URL

### TLS/SSL Errors

Use `https://` for external Loki endpoint (App Platform provides SSL).

## Limitations

### Can't View Logs with `docker logs`

When using Loki driver, `docker logs` won't work:
```bash
docker logs myapp  # Shows: "Error: configured logging driver does not support reading"
```

**Solution**: Use Grafana to view logs instead.

### No Log Rotation

Loki driver sends logs directly to Loki - no local log files.

**Impact**: If Loki is down, logs may be lost (buffered temporarily).

## Advantages

✅ **No agent needed** - Direct container-to-Loki communication
✅ **Automatic labels** - Container metadata as labels
✅ **Low overhead** - No file I/O, direct network push
✅ **Simple setup** - Just configure logging driver

## Disadvantages

❌ **No `docker logs`** - Can't use standard Docker logging commands
❌ **Limited processing** - Can't parse/filter logs before sending
❌ **Network dependency** - If Loki is down, logs may be lost
❌ **Less flexible** - Can't easily change configuration per-log-line

## When to Use Docker Driver vs Promtail/Alloy

### Use Docker Driver When:
- Running containerized apps
- Want simple setup
- Don't need log preprocessing
- Can tolerate potential log loss if Loki is down

### Use Promtail/Alloy When:
- Need log parsing/filtering
- Want buffering and retry capabilities
- Collecting logs from multiple sources (files + containers)
- Need `docker logs` to work

## Additional Resources

- **Plugin Documentation**: https://grafana.com/docs/loki/latest/clients/docker-driver/
- **Docker Logging Drivers**: https://docs.docker.com/config/containers/logging/configure/
- **GitHub**: https://github.com/grafana/loki/tree/main/clients/cmd/docker-driver
