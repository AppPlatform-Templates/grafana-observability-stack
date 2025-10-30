# Production Hardening Guide

This guide covers best practices for running Grafana and Loki in production on DigitalOcean App Platform.

## Table of Contents

1. [Security](#security)
2. [Database Configuration](#database-configuration)
3. [Storage and Retention](#storage-and-retention)
4. [Performance Tuning](#performance-tuning)
5. [Scaling](#scaling)
6. [Monitoring](#monitoring)
7. [Backup and Recovery](#backup-and-recovery)
8. [Cost Optimization](#cost-optimization)

---

## Security

### 1. Admin Credentials

**Never use default passwords in production!**

```bash
# Generate a strong password
openssl rand -base64 32

# Set in App Platform environment variables
GF_SECURITY_ADMIN_PASSWORD=<generated_strong_password>
```

### 2. Secret Key

The secret key is used for signing cookies and encrypting data.

```bash
# Generate a secret key
openssl rand -base64 64

# Set in environment
GF_SECURITY_SECRET_KEY=<generated_secret_key>
```

**Important**: Never commit secrets to Git! Use App Platform's secret management.

### 3. Disable Anonymous Access

```bash
GF_AUTH_ANONYMOUS_ENABLED=false
GF_USERS_ALLOW_SIGN_UP=false
```

### 4. Enable HTTPS Only

App Platform provides HTTPS automatically, but ensure:

```bash
GF_SERVER_PROTOCOL=http  # App Platform terminates TLS
GF_SERVER_ROOT_URL=${APP_URL}  # Full URL with https://
```

### 5. Spaces Access Keys

Protect your DigitalOcean Spaces credentials:

```bash
# Use dedicated Spaces keys with minimal permissions
# Restrict to specific bucket
SPACES_ACCESS_KEY=<restricted_key>
SPACES_SECRET_KEY=<restricted_secret>
```

**Best Practice**: Create a Spaces key specifically for Loki with:
- Read/Write access to the Loki bucket only
- No access to other buckets

### 6. Database Security

**For development database**:
- Change from `production: false` to `production: true` in AppSpec
- Enable trusted sources (restricts access to your app only)

**For managed database**:
- Enable VPC networking
- Use connection pooling
- Restrict firewall rules

### 7. Rate Limiting

Configure Loki to prevent abuse:

```yaml
# In loki/config.yaml
limits_config:
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20
  max_streams_per_user: 10000
  max_global_streams_per_user: 50000
```

### 8. Multi-Tenancy (Optional)

Enable multi-tenancy for isolation:

```yaml
# In loki/config.yaml
auth_enabled: true

# Clients must send X-Scope-OrgID header
# Example: X-Scope-OrgID: tenant1
```

Then configure Grafana:
```bash
# Grafana datasource with tenant header
GF_DATASOURCE_LOKI_JSONDATA_HTTPHEADERNAME1="X-Scope-OrgID"
GF_DATASOURCE_LOKI_SECREJSONDATA_HTTPHEADERVALUE1="tenant1"
```

---

## Database Configuration

### Development vs Production Database

**Development Database ($7/month)**:
- PostgreSQL only
- Single database
- Good for testing
- **Not recommended for production**

**Managed Database ($15+/month)**:
- Full features (HA, backups, monitoring)
- Scalable
- **Recommended for production**

### Upgrading to Production Database

1. **In `.do/deploy.template.yaml`**, change:
   ```yaml
   databases:
     - name: db
       engine: PG
       version: "16"
       production: true  # Change to true
       num_nodes: 2      # Add for HA
       size: db-s-2vcpu-4gb  # Production size
   ```

2. **Enable connection pooling**:
   ```bash
   GF_DATABASE_URL=${db.DATABASE_URL}?pool_max_conns=20
   ```

### Session Storage

Store sessions in PostgreSQL for persistence:

```bash
GF_SESSION_PROVIDER=postgres
GF_SESSION_CONN_STR=${db.DATABASE_URL}
```

**Why**: Prevents session loss on container restart.

### Database Backup

Managed databases include automatic backups:
- Daily automated backups (retained 7 days)
- Point-in-time recovery
- Manual snapshots available

**Best Practice**: Take manual snapshot before major upgrades.

---

## Storage and Retention

### Loki Storage in Spaces

Your logs are stored in DigitalOcean Spaces (S3-compatible object storage).

**Costs**:
- $5/month for 250 GB storage
- 1 TB outbound transfer included
- Additional storage: $0.02/GB/month

### Retention Policy

Configure how long logs are kept:

```yaml
# In loki/config.yaml
limits_config:
  retention_period: 744h  # 31 days

compactor:
  retention_enabled: true
  retention_delete_delay: 2h
```

**Common retention periods**:
- Development: 168h (7 days)
- Production: 744h (31 days)
- Compliance: 2160h (90 days) or longer

### Spaces Lifecycle Policy

Set up automatic deletion in Spaces:

1. Go to DigitalOcean Control Panel → Spaces
2. Select your Loki bucket
3. Settings → Lifecycle Policy
4. Add rule to delete objects older than retention period

Example:
```
Rule: Delete logs older than 90 days
Prefix: loki_index_
Days: 90
```

### Storage Monitoring

Monitor your Spaces usage:

```bash
# Check bucket size
doctl spaces ls --region nyc3 | grep loki
```

**Alert when**:
- Storage grows unexpectedly fast
- Storage exceeds budget threshold
- Outbound transfer spikes

---

## Performance Tuning

### 1. Query Performance

**Loki Configuration**:
```yaml
query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 500  # Increase for better caching

limits_config:
  split_queries_by_interval: 15m  # Split large queries
  max_cache_freshness_per_query: 10m
```

**Query Best Practices**:
- Use specific label filters
- Limit time ranges
- Avoid regex on high-cardinality labels
- Use metric queries instead of log queries when possible

### 2. Ingestion Performance

**Batch Size**:
```yaml
# In Promtail/Alloy config
batch_size: 1048576  # 1MB batches
batch_wait: 1s
```

**Loki Limits**:
```yaml
limits_config:
  ingestion_rate_mb: 50  # Increase for high volume
  ingestion_burst_size_mb: 100
  max_line_size: 256000  # 256KB per log line
```

### 3. Instance Sizing

**Current Setup (Default)**:
- Loki: `professional-xs` (1 vCPU, 2GB RAM) - $12/month
- Grafana: `professional-xs` (1 vCPU, 2GB RAM) - $12/month

**For Higher Loads**:

| Log Volume | Loki Instance | Grafana Instance | Est. Cost |
|------------|---------------|------------------|-----------|
| <20GB/day | professional-xs | professional-xs | $24/mo |
| 20-100GB/day | professional-s (1 vCPU, 4GB) | professional-xs | $36/mo |
| 100-500GB/day | professional-m (2 vCPU, 8GB) | professional-s | $84/mo |
| >500GB/day | Use Simple Scalable mode | professional-s | $150+/mo |

### 4. Grafana Performance

**Increase query timeout**:
```bash
GF_DATAPROXY_TIMEOUT=300  # 5 minutes for large queries
```

**Enable caching**:
```bash
GF_REMOTE_CACHE_TYPE=redis  # If using Redis
```

**Disable unnecessary features**:
```bash
GF_ANALYTICS_REPORTING_ENABLED=false
GF_ANALYTICS_CHECK_FOR_UPDATES=false
```

---

## Scaling

### When to Scale

**Scale Loki when**:
- Ingesting >20GB logs/day
- Query latency >5 seconds
- Memory usage consistently >80%
- Ingestion lag increases

**Scale Grafana when**:
- Dashboard load time >10 seconds
- Concurrent users >50
- Memory usage >80%

### Horizontal Scaling - Grafana

Grafana can run multiple instances with shared database:

```yaml
# In .do/deploy.template.yaml
services:
  - name: grafana
    instance_count: 2  # or enable autoscaling
    autoscaling:
      min_instance_count: 2
      max_instance_count: 5
      metrics:
        cpu:
          percent: 70
```

**Requirements**:
- PostgreSQL database (shared state)
- Session storage in database
- Disable file-based features

### Horizontal Scaling - Loki (Simple Scalable Mode)

For >20GB/day, switch to Simple Scalable deployment:

**Architecture**:
```
┌─────────────┐
│   Write     │ (Distributor + Ingester)
│   Service   │ × N replicas
└─────────────┘
        │
        ▼
   ┌─────────┐
   │  Spaces │
   └─────────┘
        │
        ▼
┌─────────────┐
│    Read     │ (Query Frontend + Querier)
│   Service   │ × N replicas
└─────────────┘
        │
        ▼
┌─────────────┐
│  Backend    │ (Compactor + Index Gateway)
│  Service    │ × 1 replica
└─────────────┘
```

**Implementation**:
1. Create 3 separate Dockerfiles (write, read, backend)
2. Use `-target=write`, `-target=read`, `-target=backend`
3. Add Nginx for routing
4. Configure ring with memberlist

See `examples/simple-scalable/` for complete setup.

### Vertical Scaling

Upgrade instance size:

```yaml
services:
  - name: loki
    instance_size_slug: professional-m  # 2 vCPU, 8GB RAM
```

**When to use**:
- Simpler than horizontal scaling
- Good for medium workloads (20-100GB/day)
- Lower operational complexity

---

## Monitoring

### Monitor Your Observability Stack

**Key Metrics to Track**:

1. **Loki Health**:
   - Ingestion rate: `loki_ingester_bytes_received_total`
   - Query latency: `loki_request_duration_seconds`
   - Memory usage: `process_resident_memory_bytes`
   - Active streams: `loki_ingester_streams`

2. **Grafana Health**:
   - Active users: `grafana_stat_active_users`
   - Dashboard load time: `grafana_api_response_time`
   - Database connections: `grafana_database_conn_*`

3. **Spaces Storage**:
   - Bucket size (via DO API)
   - Outbound transfer
   - Request rate

### Setting Up Self-Monitoring

**Add Prometheus for metrics**:
1. Deploy Prometheus on App Platform
2. Configure Prometheus to scrape Loki and Grafana
3. Add Prometheus as datasource in Grafana
4. Import monitoring dashboards

**Loki Metrics Endpoint**:
```
http://loki:3100/metrics
```

**Grafana Metrics Endpoint**:
```
http://grafana:3000/metrics
```

### Alerting

Set up alerts for critical issues:

**Loki Down**:
```logql
up{job="loki"} == 0
```

**High Ingestion Lag**:
```promql
loki_ingester_memory_streams > 100000
```

**Query Errors**:
```promql
rate(loki_request_duration_seconds_count{status_code=~"5.."}[5m]) > 0.1
```

**Grafana Down**:
```promql
up{job="grafana"} == 0
```

### Logging Your Logs

Configure Loki to send its own logs to itself:

```yaml
# In loki/config.yaml
server:
  log_level: info  # Use warn or error in production
```

Send Loki container logs to Loki using Docker logging driver or Promtail.

---

## Backup and Recovery

### What to Backup

1. **Grafana Database**:
   - Dashboards
   - Users and permissions
   - Data sources
   - Alert rules

2. **Grafana Configuration**:
   - Environment variables
   - Provisioning configs

3. **Loki Configuration**:
   - `loki/config.yaml`
   - Environment variables

4. **Loki Data** (in Spaces):
   - Automatically durable (S3-compatible storage)
   - No manual backup needed
   - Consider cross-region replication for DR

### Backup Strategy

**Database Backups**:
- Managed PostgreSQL: Automatic daily backups
- Manual snapshots before upgrades
- Test restore procedures quarterly

**Configuration Backup**:
- All configs are in Git (source of truth)
- Tag releases
- Document environment variables

**Disaster Recovery**:
1. Keep AppSpec in version control
2. Document all environment variables
3. Test full redeployment quarterly
4. Have runbook for common failures

### Recovery Procedures

**Grafana Failure**:
1. Check health endpoint: `/api/health`
2. Review logs: `doctl apps logs <app-id> --component grafana`
3. Check database connectivity
4. Restart service if needed
5. Restore from database backup if corrupted

**Loki Failure**:
1. Check health endpoint: `/ready`
2. Review logs
3. Verify Spaces connectivity
4. Check for disk space issues (`/loki/*` directories)
5. Restart service

**Data Loss**:
- Logs in Spaces are durable (99.999999999% durability)
- If bucket deleted: No recovery (stress importance of access controls)
- Regular testing of log ingestion and querying

---

## Cost Optimization

### Current Cost Breakdown

**Minimum Production Setup**:
- Loki service: $12/month (professional-xs)
- Grafana service: $12/month (professional-xs)
- PostgreSQL database: $15/month (managed db-s-1vcpu-1gb)
- Spaces storage: $5/month (250GB + 1TB transfer)
- **Total: ~$44/month**

### Optimization Strategies

### 1. Right-Size Instances

Start small, scale as needed:
- Monitor CPU and memory usage
- Scale up only when consistently >70% utilization
- Consider scheduled scaling (scale up during business hours)

### 2. Optimize Log Volume

**Reduce ingestion costs**:
- Filter noisy logs at source (Promtail/Alloy)
- Sample high-frequency logs
- Don't log debug level in production
- Aggregate similar log lines

Example Promtail filtering:
```yaml
pipeline_stages:
  - match:
      selector: '{job="myapp"}'
      stages:
        - drop:
            expression: ".*debug.*"
```

### 3. Retention Policy

Shorter retention = lower costs:
- Keep recent logs (7-31 days) in Loki
- Archive older logs to cheaper storage
- Compliance logs: Use Spaces lifecycle to archive to Glacier-equivalent

### 4. Query Optimization

- Cache frequent queries
- Use metric queries instead of log queries
- Limit time ranges in dashboards
- Pre-aggregate data where possible

### 5. Spaces Cost Management

- Enable compression (already enabled in Loki)
- Use lifecycle policies for old logs
- Monitor outbound transfer (first 1TB free)
- Consider multi-region strategy only if needed

### 6. Database Cost Management

**Development Database**:
- Use for non-critical workloads
- $7/month (vs $15 managed)
- PostgreSQL only
- Limited to single database

**When Dev DB is Sufficient**:
- Testing/staging environments
- Small deployments (<10 users)
- No HA requirements

### Cost Monitoring

Set up billing alerts in DigitalOcean:
1. Account → Billing → Alerts
2. Set threshold (e.g., $50/month)
3. Get notified before overspending

**Track costs monthly**:
```bash
# View current usage
doctl account get

# Monitor Spaces usage
doctl spaces ls --region nyc3
```

---

## Production Checklist

Before going to production, verify:

### Security
- [ ] Strong admin password set
- [ ] Secret key generated and set
- [ ] Anonymous access disabled
- [ ] User sign-up disabled
- [ ] Spaces keys have minimal permissions
- [ ] Database uses production configuration
- [ ] HTTPS enabled (automatic on App Platform)

### Performance
- [ ] Instance sizes appropriate for load
- [ ] Query caching configured
- [ ] Retention policy set
- [ ] Spaces lifecycle configured

### Reliability
- [ ] Health checks configured
- [ ] Alerts set up for critical services
- [ ] Database backups enabled
- [ ] Disaster recovery plan documented
- [ ] Tested failover procedures

### Monitoring
- [ ] Self-monitoring dashboards created
- [ ] Metrics being collected
- [ ] Log volume monitored
- [ ] Cost alerts configured

### Documentation
- [ ] Environment variables documented
- [ ] Runbooks created for common issues
- [ ] Team trained on using Grafana/Loki
- [ ] On-call procedures established

---

## Additional Resources

- **App Platform Best Practices**: https://docs.digitalocean.com/products/app-platform/best-practices/
- **Loki Production Guide**: https://grafana.com/docs/loki/latest/operations/
- **Grafana Enterprise**: https://grafana.com/products/enterprise/
- **Security Best Practices**: https://grafana.com/docs/grafana/latest/administration/security/

---

## Getting Help

If you need assistance:
1. Review logs: `doctl apps logs <app-id> --component <loki|grafana>`
2. Check health endpoints
3. Review this guide and HOW_TO_USE.md
4. Search Grafana Community forums
5. Open GitHub issue with detailed information
