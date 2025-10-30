# Use official Grafana Loki image
FROM grafana/loki:3.3.2

# Set working directory
WORKDIR /loki

# Copy custom configuration
COPY config.yaml /etc/loki/config.yaml

# Create necessary directories with proper permissions
USER root
RUN mkdir -p /loki/index /loki/index_cache /loki/compactor && \
    chown -R 10001:10001 /loki

# Switch back to non-root user
USER 10001

# Expose ports
EXPOSE 3100 9096

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3100/ready || exit 1

# Run Loki with custom config
ENTRYPOINT ["/usr/bin/loki"]
CMD ["-config.file=/etc/loki/config.yaml"]
