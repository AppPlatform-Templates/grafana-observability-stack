# Environment Variables Reference

This document lists all environment variables required and optional for deploying the Grafana Observability Stack.

## Required Variables

These variables **must** be set before deployment.

### Loki - DigitalOcean Spaces Configuration

| Variable | Description | Example | Type |
|----------|-------------|---------|------|
| `SPACES_ENDPOINT` | DigitalOcean Spaces endpoint for your region | `nyc3.digitaloceanspaces.com` | GENERAL |
| `SPACES_BUCKET` | Name of the Spaces bucket for storing logs | `my-loki-logs` | GENERAL |
| `SPACES_ACCESS_KEY` | Spaces access key ID | `DO00ABC123...` | SECRET |
| `SPACES_SECRET_KEY` | Spaces secret access key | `xyz789...` | SECRET |

**How to get Spaces credentials**:
1. Go to DigitalOcean Control Panel → API → Spaces Keys
2. Click "Generate New Key"
3. Save the access key and secret key
4. Create a new bucket: Spaces → Create Bucket
5. Use the bucket name and region endpoint

**Regions and endpoints**:
- NYC3: `nyc3.digitaloceanspaces.com`
- SFO3: `sfo3.digitaloceanspaces.com`
- AMS3: `ams3.digitaloceanspaces.com`
- SGP1: `sgp1.digitaloceanspaces.com`
- FRA1: `fra1.digitaloceanspaces.com`
- SYD1: `syd1.digitaloceanspaces.com`

---

### Grafana - Security Configuration

| Variable | Description | Example | Type |
|----------|-------------|---------|------|
| `GF_SECURITY_ADMIN_PASSWORD` | Admin user password (CHANGE THIS!) | `SuperSecret123!` | SECRET |
| `GF_SECURITY_SECRET_KEY` | Secret key for signing cookies | `SW2YcwTIb9zpOOhoPsMm` | SECRET |

**Generate secure values**:
```bash
# Generate admin password
openssl rand -base64 32

# Generate secret key
openssl rand -base64 64
```

**Important**: Never use default or simple passwords in production!

---

## Auto-Configured Variables

These variables are automatically set by App Platform and referenced in the configuration.

### Database Connection

| Variable | Description | Auto-Generated |
|----------|-------------|----------------|
| `${db.DATABASE_URL}` | PostgreSQL connection string | Yes |

Format: `postgresql://username:password@host:port/database?sslmode=require`

### App Platform Variables

| Variable | Description | Auto-Generated |
|----------|-------------|----------------|
| `${APP_URL}` | Full app URL with https:// | Yes |
| `${APP_DOMAIN}` | App domain without protocol | Yes |

---

## Optional Variables

### Grafana - General Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `GF_SERVER_ROOT_URL` | `${APP_URL}` | Full URL to access Grafana |
| `GF_SERVER_DOMAIN` | `${APP_DOMAIN}` | Domain name |
| `GF_SERVER_PROTOCOL` | `http` | Protocol (App Platform handles HTTPS) |
| `GF_SECURITY_ADMIN_USER` | `admin` | Admin username |
| `GF_DATABASE_TYPE` | `postgres` | Database type |
| `GF_SESSION_PROVIDER` | `postgres` | Session storage provider |
| `GF_SESSION_CONN_STR` | `${db.DATABASE_URL}` | Session database connection |

### Grafana - Authentication

| Variable | Default | Description |
|----------|---------|-------------|
| `GF_AUTH_ANONYMOUS_ENABLED` | `false` | Enable anonymous access |
| `GF_AUTH_ANONYMOUS_ORG_ROLE` | `Viewer` | Role for anonymous users |
| `GF_USERS_ALLOW_SIGN_UP` | `false` | Allow user registration |
| `GF_USERS_AUTO_ASSIGN_ORG` | `true` | Auto-assign users to org |
| `GF_USERS_AUTO_ASSIGN_ORG_ROLE` | `Viewer` | Default role for new users |

### Grafana - SMTP (Email Alerts)

| Variable | Default | Description |
|----------|---------|-------------|
| `GF_SMTP_ENABLED` | `false` | Enable email alerts |
| `GF_SMTP_HOST` | - | SMTP server host:port |
| `GF_SMTP_USER` | - | SMTP username |
| `GF_SMTP_PASSWORD` | - | SMTP password (SECRET) |
| `GF_SMTP_FROM_ADDRESS` | `admin@grafana.localhost` | From email address |
| `GF_SMTP_FROM_NAME` | `Grafana` | From name |

**Example SMTP Configuration**:
```
GF_SMTP_ENABLED=true
GF_SMTP_HOST=smtp.gmail.com:587
GF_SMTP_USER=your-email@gmail.com
GF_SMTP_PASSWORD=your-app-password
GF_SMTP_FROM_ADDRESS=alerts@example.com
GF_SMTP_FROM_NAME=Grafana Alerts
```

### Grafana - Plugins

| Variable | Default | Description |
|----------|---------|-------------|
| `GF_INSTALL_PLUGINS` | `""` | Comma-separated list of plugins to install |

**Example**:
```
GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-simple-json-datasource
```

**Popular plugins**:
- `grafana-clock-panel` - Clock panel
- `grafana-piechart-panel` - Pie chart panel
- `grafana-worldmap-panel` - World map panel
- `grafana-polystat-panel` - Polystat panel

Browse all plugins: https://grafana.com/grafana/plugins/

**Note**: Plugins installed this way persist across deployments. For many plugins, consider building a custom Docker image.

### Grafana - Performance

| Variable | Default | Description |
|----------|---------|-------------|
| `GF_DATAPROXY_TIMEOUT` | `30` | Data proxy timeout (seconds) |
| `GF_DATAPROXY_KEEP_ALIVE_SECONDS` | `30` | Keep-alive duration |
| `GF_RENDERING_SERVER_URL` | - | External rendering service URL |
| `GF_RENDERING_CALLBACK_URL` | - | Callback URL for rendering |

### Grafana - Logging

| Variable | Default | Description |
|----------|---------|-------------|
| `GF_LOG_MODE` | `console` | Log output mode |
| `GF_LOG_LEVEL` | `info` | Log level (debug, info, warn, error) |

**Production recommendation**: Use `info` or `warn` to reduce noise.

### Grafana - Analytics

| Variable | Default | Description |
|----------|---------|-------------|
| `GF_ANALYTICS_REPORTING_ENABLED` | `false` | Send usage stats to Grafana Labs |
| `GF_ANALYTICS_CHECK_FOR_UPDATES` | `false` | Check for Grafana updates |

---

## Configuration Examples

### Minimal Production Setup

```bash
# Loki - Spaces
SPACES_ENDPOINT=nyc3.digitaloceanspaces.com
SPACES_BUCKET=my-company-logs
SPACES_ACCESS_KEY=DO00ABC123...
SPACES_SECRET_KEY=xyz789...

# Grafana - Security
GF_SECURITY_ADMIN_PASSWORD=<generated_strong_password>
GF_SECURITY_SECRET_KEY=<generated_secret_key>

# Auto-configured by App Platform
GF_DATABASE_URL=${db.DATABASE_URL}
GF_SERVER_ROOT_URL=${APP_URL}
```

### Production with Email Alerts

```bash
# Loki - Spaces
SPACES_ENDPOINT=nyc3.digitaloceanspaces.com
SPACES_BUCKET=my-company-logs
SPACES_ACCESS_KEY=DO00ABC123...
SPACES_SECRET_KEY=xyz789...

# Grafana - Security
GF_SECURITY_ADMIN_PASSWORD=<generated_strong_password>
GF_SECURITY_SECRET_KEY=<generated_secret_key>

# Grafana - SMTP
GF_SMTP_ENABLED=true
GF_SMTP_HOST=smtp.sendgrid.net:587
GF_SMTP_USER=apikey
GF_SMTP_PASSWORD=<sendgrid_api_key>
GF_SMTP_FROM_ADDRESS=alerts@mycompany.com
GF_SMTP_FROM_NAME=Observability Alerts

# Auto-configured
GF_DATABASE_URL=${db.DATABASE_URL}
GF_SERVER_ROOT_URL=${APP_URL}
```

### Development/Testing Setup

```bash
# Loki - Spaces (use test bucket)
SPACES_ENDPOINT=nyc3.digitaloceanspaces.com
SPACES_BUCKET=test-loki-logs
SPACES_ACCESS_KEY=DO00TEST...
SPACES_SECRET_KEY=test789...

# Grafana - Security (simpler for testing)
GF_SECURITY_ADMIN_PASSWORD=testpassword123
GF_SECURITY_SECRET_KEY=testsecretkey123

# Grafana - Permissive (for testing only)
GF_AUTH_ANONYMOUS_ENABLED=true
GF_AUTH_ANONYMOUS_ORG_ROLE=Editor
GF_USERS_ALLOW_SIGN_UP=true

# Auto-configured
GF_DATABASE_URL=${db.DATABASE_URL}
GF_SERVER_ROOT_URL=${APP_URL}
```

---

## Setting Environment Variables in App Platform

### Via Control Panel

1. Go to your app in App Platform
2. Click on the service (Loki or Grafana)
3. Click "Edit" → "Environment Variables"
4. Add variables one by one
5. Mark sensitive values as "Secret"
6. Save and redeploy

### Via AppSpec (deploy.template.yaml)

```yaml
services:
  - name: grafana
    envs:
      - key: GF_SECURITY_ADMIN_PASSWORD
        value: ${GF_SECURITY_ADMIN_PASSWORD}
        type: SECRET
      - key: GF_SERVER_ROOT_URL
        value: ${APP_URL}
        type: GENERAL
```

### Via doctl CLI

```bash
# Add environment variable
doctl apps update <app-id> --spec app.yaml

# Set secret via CLI
doctl apps create-deployment <app-id> \
  --env "GF_SECURITY_ADMIN_PASSWORD=mysecretpassword"
```

---

## Security Best Practices

### Do's ✅

- **Use strong passwords** (>20 characters, mixed case, numbers, symbols)
- **Generate unique secret keys** for each environment
- **Mark sensitive values as SECRET** in App Platform
- **Rotate credentials regularly** (quarterly)
- **Use dedicated Spaces keys** with minimal permissions
- **Document variables** in this file and version control
- **Use different credentials** for dev/staging/production

### Don'ts ❌

- **Never commit secrets** to Git
- **Never use default passwords** (`admin`, `password`, etc.)
- **Never share credentials** between environments
- **Never reuse passwords** across services
- **Never log sensitive values**
- **Never store secrets** in plain text files

---

## Troubleshooting

### "Unable to connect to database"

Check:
- `GF_DATABASE_URL` is set correctly
- Database service is running
- Database allows connections from Grafana service

### "Failed to initialize Spaces storage"

Check:
- `SPACES_ENDPOINT`, `SPACES_BUCKET`, `SPACES_ACCESS_KEY`, `SPACES_SECRET_KEY` are all set
- Bucket exists in specified region
- Access keys have read/write permissions
- Endpoint matches bucket region

### "Login failed"

Check:
- `GF_SECURITY_ADMIN_PASSWORD` is set
- Using correct username (`admin` by default)
- No typos in password

### "Session expired immediately"

Check:
- `GF_SECURITY_SECRET_KEY` is set
- `GF_SESSION_PROVIDER` is set to `postgres`
- `GF_SESSION_CONN_STR` points to database

---

## Additional Resources

- **App Platform Environment Variables**: https://docs.digitalocean.com/products/app-platform/how-to/use-environment-variables/
- **Grafana Configuration**: https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/
- **Loki Configuration**: https://grafana.com/docs/loki/latest/configure/
- **Spaces API**: https://docs.digitalocean.com/reference/api/spaces-api/

---

## Need Help?

If environment variables are not working as expected:

1. Check the App Platform logs: `doctl apps logs <app-id>`
2. Verify variable names (case-sensitive!)
3. Ensure secrets are marked as SECRET type
4. Check for typos in variable values
5. Review this documentation
6. Open an issue with sanitized configuration
