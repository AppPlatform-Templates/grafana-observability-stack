# Use official Grafana image
FROM grafana/grafana:11.4.0

# Set user to root to make changes
USER root

# Copy datasource provisioning configuration
COPY provisioning/datasources/loki.yaml /etc/grafana/provisioning/datasources/loki.yaml

# Create necessary directories
RUN mkdir -p /var/lib/grafana/plugins /var/log/grafana && \
    chown -R 472:0 /var/lib/grafana /var/log/grafana /etc/grafana/provisioning

# Switch back to grafana user
USER 472

# Expose Grafana port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/api/health || exit 1

# Default command
CMD ["/run.sh"]
