# Grafana Observability Stack for DigitalOcean App Platform

[![Deploy to DO](https://www.deploytodo.com/do-btn-blue.svg)](https://cloud.digitalocean.com/apps/new?repo=https://github.com/AppPlatform-Templates/grafana-observability-stack/tree/main)

A complete observability solution combining **Grafana** for visualization and **Loki** for log aggregation, optimized for DigitalOcean App Platform.

## Features

- **Grafana 11.4.0** - Modern, feature-rich dashboarding and visualization
- **Loki 3.3.2** - Cost-effective log aggregation with label-based indexing
- **PostgreSQL Database** - Persistent storage for Grafana dashboards and users
- **DigitalOcean Spaces** - S3-compatible object storage for logs
- **Internal Networking** - Secure communication between services
- **Health Checks** - Automatic monitoring and restart
- **One-Click Deploy** - Deploy directly from this repository

## Architecture

```
┌─────────────────────────────────────────────┐
│     DigitalOcean App Platform               │
│                                             │
│  ┌──────────┐         ┌──────────┐         │
│  │  Loki    │◄────────│ Grafana  │         │
│  │  :3100   │         │  :3000   │◄────────┼─── Users (HTTPS)
│  └────┬─────┘         └────┬─────┘         │
│       │                    │               │
│       │                    ▼               │
│       │            ┌──────────────┐        │
│       │            │  PostgreSQL  │        │
│       │            └──────────────┘        │
│       │                                    │
│       ▼                                    │
│  ┌─────────────────┐                       │
│  │ DO Spaces       │                       │
│  │ (Log Storage)   │                       │
│  └─────────────────┘                       │
└─────────────────────────────────────────────┘
```

## Quick Start

### Prerequisites

1. **DigitalOcean Account** with billing enabled
2. **DigitalOcean Spaces bucket** for log storage
3. **Spaces API Keys** (access key and secret key)

### Deploy Now

Click the Deploy to DO button above, then:

1. **Connect your GitHub account** (if prompted)
2. **Configure environment variables**:
   - `SPACES_ENDPOINT` - Your Spaces region endpoint (e.g., `nyc3.digitaloceanspaces.com`)
   - `SPACES_BUCKET` - Name of your Spaces bucket
   - `SPACES_ACCESS_KEY` - Your Spaces access key (mark as SECRET)
   - `SPACES_SECRET_KEY` - Your Spaces secret key (mark as SECRET)
   - `GF_SECURITY_ADMIN_PASSWORD` - Strong password for Grafana admin (mark as SECRET)
   - `GF_SECURITY_SECRET_KEY` - Random secret key for Grafana (mark as SECRET)
3. **Click "Deploy"**
4. Wait 5-10 minutes for deployment to complete
5. Access your Grafana instance at the provided App URL

### Creating a Spaces Bucket

1. Go to [DigitalOcean Control Panel → Spaces](https://cloud.digitalocean.com/spaces)
2. Click **"Create Bucket"**
3. Choose a **region** (e.g., NYC3, SFO3)
4. Name your bucket (e.g., `my-company-loki-logs`)
5. Set permissions to **Private**
6. Click **"Create Bucket"**

### Generating Spaces API Keys

1. Go to [API → Spaces Keys](https://cloud.digitalocean.com/account/api/spaces)
2. Click **"Generate New Key"**
3. Name it (e.g., "Loki Storage")
4. **Save both the Access Key and Secret Key** immediately (you won't see the secret again)

### Generating Secure Passwords

```bash
# Generate admin password
openssl rand -base64 32

# Generate secret key
openssl rand -base64 64
```

## What's Included

### Grafana Service

- **Purpose**: Visualization, dashboarding, and alerting
- **Port**: 3000 (HTTPS via App Platform)
- **Database**: PostgreSQL for dashboards and user management
- **Datasource**: Pre-configured Loki connection
- **Health Check**: `/api/health`

### Loki Service

- **Purpose**: Log aggregation and querying
- **Port**: 3100 (internal only)
- **Storage**: DigitalOcean Spaces (S3-compatible)
- **Mode**: Monolithic (good for up to 20GB/day)
- **Health Check**: `/ready`

### PostgreSQL Database

- **Purpose**: Grafana state (dashboards, users, sessions)
- **Engine**: PostgreSQL 16
- **Size**: Development database (can upgrade to production)

## Documentation

- **[How to Use](docs/HOW_TO_USE.md)** - Complete getting started guide
  - First login and setup
  - Sending logs to Loki
  - Creating dashboards
  - Setting up alerts
  - Common LogQL queries

- **[Environment Variables](docs/ENV_TEMPLATE.md)** - All configuration options
  - Required variables
  - Optional settings
  - Security best practices
  - Configuration examples

- **[Production Hardening](docs/PRODUCTION_HARDENING.md)** - Production deployment guide
  - Security hardening
  - Performance tuning
  - Scaling strategies
  - Cost optimization
  - Backup and recovery

- **[Version Information](docs/VERSION.md)** - Component versions and upgrade guide
  - Current versions
  - Update policy
  - Upgrade procedures

## Pricing

### Minimum Configuration (~$44/month)

| Component | Instance | Cost |
|-----------|----------|------|
| Loki | professional-xs (1 vCPU, 2GB RAM) | $12/mo |
| Grafana | professional-xs (1 vCPU, 2GB RAM) | $12/mo |
| PostgreSQL | db-s-1vcpu-1gb (Dev or Managed) | $7-15/mo |
| Spaces | 250GB storage + 1TB transfer | $5/mo |
| **Total** | | **$36-44/mo** |

### Scaling Costs

For higher log volumes or traffic:

| Log Volume | Configuration | Est. Cost |
|------------|--------------|-----------|
| <20GB/day | Default (2x professional-xs) | $36-44/mo |
| 20-100GB/day | Loki: professional-s, Grafana: professional-xs | $48-56/mo |
| 100-500GB/day | Loki: professional-m, Grafana: professional-s | $96-104/mo |
| >500GB/day | Simple Scalable mode (3+ services) | $150+/mo |

**Note**: Spaces costs scale with usage ($0.02/GB/month for storage over 250GB).

## Log Forwarding

Loki doesn't collect logs automatically. You need to send logs using an agent:

### Option 1: Promtail (Official Agent)

```yaml
# promtail-config.yaml
clients:
  - url: https://your-app.ondigitalocean.app/loki/api/v1/push
    # Or internal: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log
```

See `examples/promtail/` for complete setup.

### Option 2: Grafana Alloy (New Agent)

```hcl
loki.write "default" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

See `examples/alloy/` for complete setup.

### Option 3: Docker Logging Driver

```yaml
services:
  myapp:
    logging:
      driver: loki
      options:
        loki-url: "http://loki:3100/loki/api/v1/push"
```

## First Login

1. Navigate to your App URL: `https://your-app-name.ondigitalocean.app`
2. Login with:
   - Username: `admin`
   - Password: (value you set for `GF_SECURITY_ADMIN_PASSWORD`)
3. **Change your password immediately** via Profile → Change Password

## Sample Queries

Once logs are flowing to Loki, try these queries in Grafana's Explore page:

```logql
# View all logs
{job="myapp"}

# Filter by content
{job="myapp"} |= "error"

# Regex match
{job="myapp"} |~ "error|warning"

# Count over time
count_over_time({job="myapp"}[5m])

# Rate of errors
rate({job="myapp"} |= "error" [5m])

# Parse JSON logs
{job="myapp"} | json | level="error"
```

## Common Use Cases

### Application Monitoring

- Aggregate logs from all your applications
- Filter by service, environment, or region
- Track error rates and patterns
- Set up alerts for critical issues

### Debugging and Troubleshooting

- Live tail logs in real-time
- Search historical logs
- Correlate errors across services
- Trace request flows

### Security and Compliance

- Centralized audit logging
- Retain logs for compliance (configurable retention)
- Alert on suspicious patterns
- Secure storage with encryption

### DevOps and SRE

- Service health monitoring
- Performance analysis
- Incident response
- Post-mortem analysis

## Limitations

### Current Limitations

- **Ephemeral local storage**: Loki uses 2GB ephemeral disk for index caching
- **Single instance**: Monolithic mode runs single Loki instance
- **Log volume**: Best for <20GB/day (can scale to Simple Scalable mode)
- **No automatic log collection**: Requires log agent setup

### When to Consider Alternatives

- **>500GB logs/day**: Consider Grafana Cloud or self-hosted Kubernetes
- **Multi-region**: Requires custom setup with multiple deployments
- **High availability**: Requires Simple Scalable or Microservices mode

## Scaling

### Horizontal Scaling - Grafana

Grafana supports multiple instances with shared PostgreSQL:

```yaml
services:
  - name: grafana
    instance_count: 2
    # Or enable autoscaling
```

### Scaling Loki

For >20GB/day, upgrade to Simple Scalable deployment mode:
- Separate read, write, and backend services
- Independent scaling of each component
- See `examples/simple-scalable/` for configuration

## Troubleshooting

### Loki Not Receiving Logs

1. Check Loki health: `curl https://your-app.app/loki/ready`
2. Verify Spaces credentials are correct
3. Check log agent configuration (Promtail/Alloy)
4. Review Loki logs: `doctl apps logs <app-id> --component loki`

### Grafana Login Issues

1. Verify `GF_SECURITY_ADMIN_PASSWORD` is set
2. Check for typos in username/password
3. Review Grafana logs: `doctl apps logs <app-id> --component grafana`

### Database Connection Errors

1. Ensure PostgreSQL database is running
2. Verify `${db.DATABASE_URL}` is being populated
3. Check database firewall/trusted sources settings

### Spaces Connection Errors

1. Verify Spaces bucket exists in specified region
2. Check Spaces keys have read/write permissions
3. Ensure `SPACES_ENDPOINT` matches bucket region
4. Test access with AWS CLI or similar tool

## Support and Contributing

### Getting Help

- **Documentation**: See `docs/` folder
- **Issues**: [GitHub Issues](https://github.com/AppPlatform-Templates/grafana-observability-stack/issues)
- **Community**: [Grafana Community Forums](https://community.grafana.com/)
- **App Platform**: [DigitalOcean Community](https://www.digitalocean.com/community/tags/app-platform)

### Contributing

Contributions are welcome! Please:
1. Fork this repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

### Reporting Issues

When reporting issues, please include:
- App Platform logs
- Environment variable configuration (sanitized)
- Steps to reproduce
- Expected vs actual behavior

## License

This repository is provided as-is under the MIT License. Grafana and Loki are licensed under AGPL-3.0.

## Acknowledgments

- **Grafana Labs** for Grafana and Loki
- **DigitalOcean** for App Platform
- **Community contributors** for feedback and improvements

---

## Quick Links

- **Grafana Documentation**: https://grafana.com/docs/grafana/latest/
- **Loki Documentation**: https://grafana.com/docs/loki/latest/
- **LogQL Guide**: https://grafana.com/docs/loki/latest/logql/
- **App Platform Docs**: https://docs.digitalocean.com/products/app-platform/
- **DigitalOcean Spaces**: https://docs.digitalocean.com/products/spaces/

---

**Ready to get started?** Click the Deploy to DO button at the top of this page!
