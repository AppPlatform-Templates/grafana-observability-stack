# Version Information

This repository packages Grafana and Loki together for deployment on DigitalOcean App Platform.

## Current Versions

### Grafana
- **Version**: 11.4.0
- **Release Date**: December 2024
- **Official Image**: `grafana/grafana:11.4.0`
- **Upstream Repository**: https://github.com/grafana/grafana
- **Release Notes**: https://github.com/grafana/grafana/releases/tag/v11.4.0

### Loki
- **Version**: 3.3.2
- **Release Date**: December 2024
- **Official Image**: `grafana/loki:3.3.2`
- **Upstream Repository**: https://github.com/grafana/loki
- **Release Notes**: https://github.com/grafana/loki/releases/tag/v3.3.2

## Version Policy

### Update Strategy
This repository tracks stable releases of both Grafana and Loki. We update to:
- ✅ New minor versions (e.g., 11.4.x → 11.5.0)
- ✅ Patch releases (e.g., 11.4.0 → 11.4.1)
- ⚠️ Major versions after thorough testing (e.g., 11.x → 12.0)

### Compatibility
The versions included in this repository have been tested together and are known to be compatible. When updating:
- Grafana supports Loki versions within ±1 major version
- Loki API is backward compatible within major versions
- Configuration format may change between major versions

## Checking for Updates

### Grafana Updates
Check for new Grafana releases:
```bash
# Via Docker Hub
docker pull grafana/grafana:latest
docker inspect grafana/grafana:latest | grep -i version

# Via GitHub
curl -s https://api.github.com/repos/grafana/grafana/releases/latest | grep tag_name
```

Visit: https://github.com/grafana/grafana/releases

### Loki Updates
Check for new Loki releases:
```bash
# Via Docker Hub
docker pull grafana/loki:latest
docker inspect grafana/loki:latest | grep -i version

# Via GitHub
curl -s https://api.github.com/repos/grafana/loki/releases/latest | grep tag_name
```

Visit: https://github.com/grafana/loki/releases

## Upgrading

### Updating Grafana
1. Update the version in `grafana/Dockerfile`:
   ```dockerfile
   FROM grafana/grafana:11.5.0  # Change version here
   ```

2. Review release notes for breaking changes
3. Test locally with `docker-compose`
4. Deploy to test environment
5. Verify dashboards and datasources work
6. Update production

### Updating Loki
1. Update the version in `loki/Dockerfile`:
   ```dockerfile
   FROM grafana/loki:3.4.0  # Change version here
   ```

2. Review release notes for configuration changes
3. Update `loki/config.yaml` if needed
4. Test log ingestion and queries
5. Verify storage compatibility
6. Update production

### Database Migrations
Grafana automatically handles database migrations on startup. When upgrading:
- ✅ Backup your Grafana PostgreSQL database first
- ✅ Allow extra startup time for migrations
- ✅ Monitor logs during first startup after upgrade

### Configuration Changes
Configuration format may change between versions:
- Review `loki/config.yaml` against latest examples
- Check Grafana environment variables for deprecations
- Update App Platform AppSpec if needed

## Version History

| Date | Grafana | Loki | Notes |
|------|---------|------|-------|
| 2025-01 | 11.4.0 | 3.3.2 | Initial release for App Platform |

## Support

### Upstream Support
- Grafana: https://community.grafana.com/
- Loki: https://github.com/grafana/loki/discussions

### App Platform Support
- Documentation: https://docs.digitalocean.com/products/app-platform/
- Community: https://www.digitalocean.com/community/tags/app-platform

## Long-Term Support (LTS)

### Grafana LTS
Grafana does not have traditional LTS releases, but maintains:
- Security patches for current release
- Bug fixes for current + previous major version
- Breaking changes documented in migration guides

### Loki Versioning
Loki follows semantic versioning:
- Major versions may include breaking changes
- Minor versions add features, maintain compatibility
- Patch versions are bug fixes only

## Deprecation Notice

Watch for deprecations in:
- Configuration options
- API endpoints
- Storage format changes
- Authentication methods

Check release notes before upgrading major versions.
