# How to Use Grafana Observability Stack

This guide will help you get started with your Grafana + Loki observability stack on DigitalOcean App Platform.

## Table of Contents

1. [First Login](#first-login)
2. [Verifying Loki Connection](#verifying-loki-connection)
3. [Sending Logs to Loki](#sending-logs-to-loki)
4. [Exploring Logs](#exploring-logs)
5. [Creating Dashboards](#creating-dashboards)
6. [Setting Up Alerts](#setting-up-alerts)
7. [Common LogQL Queries](#common-logql-queries)

---

## First Login

### Accessing Grafana

After deployment, your Grafana instance will be available at your App Platform URL.

1. Open your browser and navigate to your app URL (e.g., `https://your-app-name.ondigitalocean.app`)

2. You'll see the Grafana login page

3. Log in with your credentials:
   - **Username**: `admin`
   - **Password**: The value you set for `GF_SECURITY_ADMIN_PASSWORD`

4. **Important**: Change your password immediately after first login
   - Click on your profile icon (bottom left)
   - Select "Profile"
   - Click "Change password"

### Dashboard Overview

After login, you'll see the Grafana home page with:
- **Navigation menu** (left sidebar)
- **Dashboards** - View and create dashboards
- **Explore** - Ad-hoc log queries and exploration
- **Alerting** - Set up and manage alerts
- **Configuration** - Data sources, users, plugins

---

## Verifying Loki Connection

### Check Loki Data Source

1. Navigate to **Configuration** → **Data sources** (gear icon in left sidebar)

2. You should see **Loki** listed as a data source
   - Status: **✓ Data source is working**
   - URL: `http://loki:3100`

3. If not connected, check:
   - Both Loki and Grafana services are running
   - Loki health endpoint: Access Loki's `/ready` endpoint
   - Internal networking between services

### Test Loki Connection

1. Go to **Explore** (compass icon in left sidebar)

2. Select **Loki** from the data source dropdown

3. Try a simple query:
   ```logql
   {job="loki"}
   ```

4. If you see logs, Loki is working correctly

---

## Sending Logs to Loki

Loki doesn't collect logs automatically - you need to send logs to it using a log agent.

### Option 1: Using Promtail (Recommended)

Promtail is the official log agent for Loki.

**Docker Compose Example**:
```yaml
services:
  promtail:
    image: grafana/promtail:latest
    volumes:
      - ./promtail-config.yaml:/etc/promtail/config.yaml
      - /var/log:/var/log:ro
    command: -config.file=/etc/promtail/config.yaml
```

**Promtail Configuration** (`promtail-config.yaml`):
```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: https://your-app-name.ondigitalocean.app/loki/api/v1/push
    # For internal access (if Promtail is in same app):
    # - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log
```

See `examples/promtail/` directory for complete examples.

### Option 2: Using Grafana Alloy

Grafana Alloy is the new, more flexible agent.

**Configuration Example**:
```hcl
loki.write "default" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}

loki.source.file "logs" {
  targets = [
    {__path__ = "/var/log/*.log", job = "system"},
  ]
  forward_to = [loki.write.default.receiver]
}
```

See `examples/alloy/` directory for complete examples.

### Option 3: Using Docker Logging Driver

For containerized applications:

```yaml
services:
  myapp:
    image: myapp:latest
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
        loki-batch-size: "400"
        labels: "job=myapp,environment=production"
```

### Option 4: Direct HTTP Push

Send logs directly via HTTP POST:

```bash
curl -X POST http://loki:3100/loki/api/v1/push \
  -H "Content-Type: application/json" \
  -d '{
    "streams": [
      {
        "stream": {
          "job": "test",
          "level": "info"
        },
        "values": [
          ["'"$(date +%s)000000000"'", "Hello, Loki!"]
        ]
      }
    ]
  }'
```

---

## Exploring Logs

### Using the Explore Page

1. Click **Explore** in the left sidebar

2. Select **Loki** as the data source

3. Build your query using the query builder or write LogQL directly

### Basic Log Queries

**View all logs**:
```logql
{job="varlogs"}
```

**Filter by label**:
```logql
{job="myapp", environment="production"}
```

**Filter by log content**:
```logql
{job="myapp"} |= "error"
```

**Exclude log content**:
```logql
{job="myapp"} != "debug"
```

**Regular expression match**:
```logql
{job="myapp"} |~ "error|warning|critical"
```

### Time Range

- Use the time picker in the top-right corner
- Quick options: Last 5m, 15m, 1h, 6h, 24h, 7d
- Or set custom time range

### Live Tailing

Click the **Live** button to stream logs in real-time (similar to `tail -f`).

---

## Creating Dashboards

### Create Your First Dashboard

1. Click **Dashboards** → **New** → **New Dashboard**

2. Click **Add visualization**

3. Select **Loki** as the data source

4. Write a query, for example:
   ```logql
   sum by (level) (count_over_time({job="myapp"}[5m]))
   ```

5. Choose a visualization type:
   - **Time series** - For trends over time
   - **Logs** - Display raw log lines
   - **Table** - Structured data
   - **Bar chart** - Comparisons
   - **Stat** - Single value metrics

6. Configure panel options:
   - Title, description
   - Legends, axes
   - Thresholds, colors

7. Click **Apply** and **Save dashboard**

### Example Dashboard: Application Logs

**Panel 1: Log Volume Over Time**
```logql
sum(count_over_time({job="myapp"}[1m]))
```

**Panel 2: Error Rate**
```logql
sum(rate({job="myapp"} |= "error" [5m]))
```

**Panel 3: Log Levels Distribution**
```logql
sum by (level) (count_over_time({job="myapp"}[5m]))
```

**Panel 4: Recent Error Logs**
```logql
{job="myapp"} |= "error"
```
Visualization: Logs panel, limit to 50 lines

---

## Setting Up Alerts

### Create a Log-Based Alert

1. Go to **Alerting** → **Alert rules**

2. Click **New alert rule**

3. Configure the alert:

**Step 1: Set query and condition**
```logql
sum(count_over_time({job="myapp"} |= "error" [5m])) > 10
```
This triggers if more than 10 errors occur in 5 minutes.

**Step 2: Define alert condition**
- Condition: `WHEN last() OF query(A) IS ABOVE 10`
- Evaluation: `FOR 5m`

**Step 3: Add details**
- Alert name: "High Error Rate"
- Folder: "Application Alerts"
- Group: "myapp"

**Step 4: Annotations**
- Summary: "High error rate detected"
- Description: "More than 10 errors in the last 5 minutes"

**Step 5: Notifications**
- Add contact point (email, Slack, PagerDuty, etc.)

4. Click **Save and exit**

### Example Alert Queries

**High error rate**:
```logql
sum(rate({job="myapp"} |= "error" [5m])) > 0.1
```

**Service down (no logs)**:
```logql
sum(count_over_time({job="myapp"}[5m])) == 0
```

**Pattern detection**:
```logql
sum(count_over_time({job="myapp"} |~ "out of memory|timeout" [5m])) > 0
```

---

## Common LogQL Queries

### Log Stream Selection

```logql
# All logs from a job
{job="myapp"}

# Multiple labels
{job="myapp", environment="production", region="us-east"}

# Label regex
{job=~"myapp.*"}

# Exclude labels
{job="myapp", environment!="development"}
```

### Log Pipeline

```logql
# Filter for specific text
{job="myapp"} |= "error"

# Exclude text
{job="myapp"} != "debug"

# Regex match
{job="myapp"} |~ "error|warning"

# Case-insensitive
{job="myapp"} |~ "(?i)error"

# Chain filters
{job="myapp"} |= "error" |= "database" != "connection pool"
```

### Parsing Logs

```logql
# JSON parsing
{job="myapp"} | json | level="error"

# Logfmt parsing
{job="myapp"} | logfmt | status_code="500"

# Pattern parsing
{job="myapp"} | pattern `<ip> - <user> [<timestamp>] "<method> <path>" <status>`

# Regex parsing
{job="myapp"} | regexp `status=(?P<status>\d+)`
```

### Aggregations

```logql
# Count logs over time
count_over_time({job="myapp"}[5m])

# Rate of logs
rate({job="myapp"}[5m])

# Sum
sum(count_over_time({job="myapp"}[5m]))

# By label
sum by (level) (count_over_time({job="myapp"}[5m]))

# Average
avg(count_over_time({job="myapp"}[5m]))
```

### Metrics from Logs

```logql
# Extract duration and calculate average
avg_over_time({job="myapp"} | json | unwrap duration [5m])

# 95th percentile
quantile_over_time(0.95, {job="myapp"} | json | unwrap duration [5m])

# Count by extracted field
sum by (status_code) (count_over_time({job="myapp"} | json [5m]))
```

---

## Next Steps

1. **Explore the Dashboards**
   - Import community dashboards from https://grafana.com/grafana/dashboards/

2. **Set Up Alerts**
   - Configure notification channels (Slack, email, PagerDuty)
   - Create meaningful alerts for your application

3. **Optimize Performance**
   - Review `PRODUCTION_HARDENING.md` for production best practices
   - Configure log retention policies
   - Set up caching if needed

4. **Add More Data Sources**
   - Connect Prometheus for metrics
   - Add Tempo for distributed tracing
   - Complete observability stack!

---

## Troubleshooting

### No Logs Appearing

1. **Check Loki is running**:
   ```bash
   curl http://loki:3100/ready
   ```

2. **Verify logs are being sent**:
   - Check Promtail/Alloy logs
   - Test with manual HTTP push

3. **Check Loki logs**:
   ```bash
   doctl apps logs <app-id> --component loki
   ```

### Query Performance Issues

1. **Narrow time range**: Use smaller time windows
2. **Add more labels**: Use specific label filters
3. **Use metric queries**: Aggregate instead of raw logs

### Authentication Issues

1. **Reset admin password**:
   - Update `GF_SECURITY_ADMIN_PASSWORD` environment variable
   - Redeploy application

2. **Enable anonymous access** (for public dashboards):
   ```
   GF_AUTH_ANONYMOUS_ENABLED=true
   GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer
   ```

---

## Resources

- **Grafana Docs**: https://grafana.com/docs/grafana/latest/
- **Loki Docs**: https://grafana.com/docs/loki/latest/
- **LogQL Docs**: https://grafana.com/docs/loki/latest/logql/
- **Community Dashboards**: https://grafana.com/grafana/dashboards/
- **Community Forums**: https://community.grafana.com/

---

## Support

If you encounter issues:
1. Check the logs: `doctl apps logs <app-id>`
2. Review `ENV_TEMPLATE.md` for configuration
3. See `PRODUCTION_HARDENING.md` for advanced setup
4. Open an issue on GitHub: [Repository Issues](https://github.com/AppPlatform-Templates/grafana-observability-stack/issues)
