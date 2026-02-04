# Senzing ELK Stack Deployment

![CI](https://github.com/YOUR-ORG/sz_kibana_dashboard/workflows/CI/badge.svg)
![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)
![Elasticsearch](https://img.shields.io/badge/elasticsearch-9.3.0-00BFB3.svg)
![Kibana](https://img.shields.io/badge/kibana-9.3.0-F04E98.svg)
![Logstash](https://img.shields.io/badge/logstash-9.3.0-FEC514.svg)

A complete Elasticsearch, Logstash, and Kibana (ELK) stack for centralized log collection and monitoring of Senzing applications across multiple machines.

## 📚 Documentation

- **[Configuration Reference](CONFIGURATION.md)** - Complete configuration guide
- **[Security Guide](SECURITY.md)** - Production security best practices
- **[Troubleshooting](TROUBLESHOOTING.md)** - Solutions to common problems
- **[Contributing](CONTRIBUTING.md)** - How to contribute to this project

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Quick Start](#quick-start)
- [System Requirements](#system-requirements)
- [Dashboard Components](#dashboard-components)
- [Configuration](#configuration)
- [Management](#management)
- [Troubleshooting](#troubleshooting)
- [Security](#security)
- [Advanced Topics](#advanced-topics)
- [Getting Help](#getting-help)
- [License](#license)

## Overview

This deployment provides:

- ✅ **Centralized log aggregation** from multiple remote machines
- ✅ **Real-time log analysis** with Kibana dashboards
- ✅ **Automated dashboard provisioning** with pre-configured searches
- ✅ **Optimized column layouts** for efficient log viewing
- ✅ **Multi-machine GELF log collection** via Logstash
- ✅ **Production-ready** with security hardening options
- ✅ **Automated testing** via GitHub Actions CI/CD

## Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Remote Machine │    │  Remote Machine │    │       ...       │
│  192.168.2.151  │    │  192.168.2.142  │    │                 │
│                 │    │                 │    │                 │
│ Docker Containers│    │ Docker Containers│    │ Docker Containers│
│ (GELF logging)  │    │ (GELF logging)  │    │ (GELF logging)  │
└─────────┬───────┘    └─────────┬───────┘    └─────────┬───────┘
          │                      │                      │
          └──────────────────────┼──────────────────────┘
                                 │ UDP 12201 (GELF)
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ELK Stack (192.168.2.100)                   │
│                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────────┐│
│  │  Logstash   │  │Elasticsearch │  │        Kibana           ││
│  │ :12201/udp  │─▶│   :9200      │◀─│ :5601 + Dashboard       ││
│  │ (GELF input)│  │              │  │ (Senzing Dashboard)     ││
│  └─────────────┘  └──────────────┘  └─────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Version | Purpose | Ports |
|-----------|---------|---------|-------|
| **Elasticsearch** | 9.3.0 | Log storage and indexing | 9200, 9300 |
| **Logstash** | 9.3.0 | Log ingestion and processing | 12201 (UDP), 9600 |
| **Kibana** | 9.3.0 | Web UI and dashboards | 5601 |
| **Dashboard Importer** | latest | Automated dashboard setup | N/A |

## Quick Start

### Prerequisites

**Required:**
- Docker Engine 20.10+ ([Install Docker](https://docs.docker.com/get-docker/))
- Docker Compose 2.0+ ([Install Compose](https://docs.docker.com/compose/install/))
- 2GB+ available RAM (4GB+ recommended)
- 10GB+ free disk space

**Verify installation:**
```bash
docker --version
# Docker version 24.0.0, build ...

docker compose version
# Docker Compose version v2.20.0
```

**Network requirements:**
- Ports 12201/udp (GELF) and 5601/tcp (Kibana) accessible from remote machines
- Firewall rules allowing traffic between ELK server and remote machines

### Deployment

1. **Clone or download this repository**
   ```bash
   git clone https://github.com/YOUR-ORG/sz_kibana_dashboard.git
   cd sz_kibana_dashboard
   ```

2. **Deploy the ELK stack**
   ```bash
   docker compose -f kibana.yaml up -d
   ```

   **Expected output:**
   ```
   [+] Running 5/5
   ✔ Network senzing                 Created
   ✔ Container senzing-elasticsearch Started
   ✔ Container senzing-logstash      Started
   ✔ Container senzing-kibana        Started
   ✔ Container senzing-kibana-dashboard Started
   ```

3. **Verify deployment**
   ```bash
   # Check service status
   docker compose -f kibana.yaml ps

   # Check Elasticsearch health
   curl http://localhost:9200/_cluster/health?pretty

   # Check Kibana status (may take 30-60 seconds)
   curl http://localhost:5601/api/status
   ```

4. **Access Kibana Dashboard**
   - Open browser to: **http://192.168.2.100:5601** (or http://localhost:5601)
   - Navigate to: **Dashboard → "Senzing Dashboard"**

### Configure Remote Machines

On machines you want to collect logs from, configure Docker containers to send logs via GELF:

```yaml
services:
  your-app:
    image: your-app:latest
    logging:
      driver: gelf
      options:
        gelf-address: "udp://192.168.2.100:12201"
        tag: "{{.Name}}"
```

**Test log ingestion:**
```bash
# Send a test message from remote machine
echo '{"version":"1.1","host":"test","short_message":"Test log","level":6,"_container_name":"test"}' | \
  nc -u 192.168.2.100 12201

# Verify in Elasticsearch (on ELK server)
curl -s "http://localhost:9200/log-*/_search?size=1&sort=@timestamp:desc&pretty"
```

## System Requirements

### Minimum Requirements (Development)

- **CPU**: 2 cores
- **RAM**: 2GB
- **Disk**: 10GB
- **OS**: Linux, macOS, Windows with WSL2

### Recommended Requirements (Production)

- **CPU**: 4+ cores
- **RAM**: 8GB+ (16GB for heavy workloads)
- **Disk**: 100GB+ SSD
- **OS**: Linux (Ubuntu 20.04+, RHEL 8+, Debian 11+)

### Resource Allocation

Memory allocation by component:

| Component | Min RAM | Recommended RAM | Heavy Load |
|-----------|---------|-----------------|------------|
| Elasticsearch | 512MB | 1-2GB | 4GB+ |
| Logstash | 256MB | 512MB | 1GB+ |
| Kibana | 256MB | 512MB | 1GB |
| **Total** | **1GB** | **2-3GB** | **6GB+** |

See [CONFIGURATION.md](CONFIGURATION.md) for tuning guidelines.

## Dashboard Components

The **Senzing Dashboard** includes three pre-configured panels in a 2x2 grid layout:

### 1. Error Logs (Top-Left)
- **Filter**: `message: (*error* OR *Error* OR *ERROR* OR *fail* OR *exception*)`
- **Purpose**: Catch all error messages and failures
- **Columns**: `@timestamp`, `message`, `container_name`, `source_host`

### 2. Workload Statistics (Top-Right)
- **Filter**: `message: *workload*`
- **Purpose**: Monitor workload processing statistics
- **Example**: "Processed 1730000 adds, 112 records per second"

### 3. All Recent Logs (Bottom)
- **Filter**: None (all logs)
- **Purpose**: Complete log stream, most recent first
- **Sort**: `@timestamp desc`

### Dashboard Features

- **Auto-refresh**: Configure in Kibana (top-right)
- **Time range**: Adjustable time picker (last 15m, 1h, 24h, 7d, 30d)
- **Column widths**: Optimized for readability
  - `container_name`: 150px
  - `source_host`: 120px
  - `@timestamp`, `message`: Auto-sizing

## Configuration

### Environment Variables

Customize versions and network via environment variables:

```bash
# Create .env file
cat > .env << EOF
SENZING_DOCKER_IMAGE_VERSION_ELASTICSEARCH=9.3.0
SENZING_DOCKER_IMAGE_VERSION_LOGSTASH=9.3.0
SENZING_DOCKER_IMAGE_VERSION_KIBANA=9.3.0
SENZING_DOCKER_NETWORK=senzing
EOF

# Deploy with custom configuration
docker compose -f kibana.yaml up -d
```

### Memory Configuration

Adjust heap sizes for your environment:

```yaml
# Edit kibana.yaml
elasticsearch:
  environment:
    ES_JAVA_OPTS: "-Xmx1g -Xms1g"  # Default: 512m

logstash:
  environment:
    LS_JAVA_OPTS: "-Xmx512m -Xms512m"  # Default: 256m
```

**See [CONFIGURATION.md](CONFIGURATION.md) for complete configuration reference.**

## Management

### View Logs

```bash
# All services
docker compose -f kibana.yaml logs -f

# Specific service
docker compose -f kibana.yaml logs kibana
docker compose -f kibana.yaml logs logstash
docker compose -f kibana.yaml logs elasticsearch
```

### Restart Services

```bash
# Restart all services
docker compose -f kibana.yaml restart

# Restart specific service
docker compose -f kibana.yaml restart kibana
```

### Update Stack

```bash
# Pull latest images
docker compose -f kibana.yaml pull

# Recreate containers
docker compose -f kibana.yaml up -d
```

### Stop and Remove

```bash
# Stop services (keeps data)
docker compose -f kibana.yaml down

# Stop and remove data volumes (CAUTION: deletes all logs)
docker compose -f kibana.yaml down -v
```

### Backup and Restore

**Backup Elasticsearch data:**

```bash
# Create snapshot repository
curl -X PUT "http://localhost:9200/_snapshot/backup_repo" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "fs",
    "settings": {
      "location": "/usr/share/elasticsearch/backup"
    }
  }'

# Create snapshot
curl -X PUT "http://localhost:9200/_snapshot/backup_repo/snapshot_1?wait_for_completion=true"

# Copy backup files
docker cp senzing-elasticsearch:/usr/share/elasticsearch/backup ./elasticsearch-backup
```

**Restore:**

```bash
# Copy backup files to container
docker cp ./elasticsearch-backup senzing-elasticsearch:/usr/share/elasticsearch/backup

# Restore snapshot
curl -X POST "http://localhost:9200/_snapshot/backup_repo/snapshot_1/_restore"
```

## Troubleshooting

### Quick Diagnostics

```bash
# Check service status
docker compose -f kibana.yaml ps

# Check Elasticsearch health
curl http://localhost:9200/_cluster/health?pretty

# Check logs for errors
docker compose -f kibana.yaml logs --tail=100 | grep -i error

# Check disk space
df -h

# Check memory usage
docker stats --no-stream
```

### Common Issues

| Problem | Solution |
|---------|----------|
| Services won't start | Check available RAM and disk space |
| No logs appearing | Verify GELF configuration and network connectivity |
| Dashboard not loading | Check Kibana logs and verify Elasticsearch health |
| High memory usage | Reduce heap sizes in kibana.yaml |
| Disk full | Delete old indices or implement ILM |

**See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for detailed solutions.**

## Security

⚠️ **WARNING**: The default configuration is for internal/development networks and is **NOT secure** for production.

### Security Checklist for Production

- [ ] Enable Elasticsearch Security (X-Pack)
- [ ] Configure TLS/SSL for all services
- [ ] Set up authentication (users and roles)
- [ ] Restrict network access (firewall rules)
- [ ] Change default passwords
- [ ] Enable audit logging
- [ ] Implement secrets management
- [ ] Regular security updates

**See [SECURITY.md](SECURITY.md) for complete security hardening guide.**

## Advanced Topics

### Index Lifecycle Management

Automatically delete old logs:

```bash
# Delete indices older than 30 days
curl -X DELETE "http://localhost:9200/log-$(date -d '30 days ago' +%Y.%m.%d)"
```

See [CONFIGURATION.md](CONFIGURATION.md#index-lifecycle-management) for ILM policy configuration.

### Custom Dashboards

Export modified dashboards:

```bash
curl -X POST "http://localhost:5601/api/saved_objects/_export" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{"type": "dashboard", "includeReferencesDeep": true}' > dashboard-export.ndjson
```

Update NDJSON in `kibana.yaml` (lines 81-88) and redeploy.

### High Availability

For production deployments with HA:
- Deploy Elasticsearch cluster (3+ nodes)
- Load balance Kibana (2+ instances)
- Deploy multiple Logstash instances

## Getting Help

### Documentation

- [Configuration Reference](CONFIGURATION.md)
- [Security Guide](SECURITY.md)
- [Troubleshooting Guide](TROUBLESHOOTING.md)
- [Contributing Guide](CONTRIBUTING.md)

### Support Channels

- **GitHub Issues**: Report bugs or request features
- **Elasticsearch Forums**: https://discuss.elastic.co
- **Docker Forums**: https://forums.docker.com

### Before Asking for Help

1. Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
2. Search existing GitHub issues
3. Collect debug information:
   ```bash
   docker compose -f kibana.yaml logs > elk-logs.txt
   curl -s "http://localhost:9200/_cluster/health?pretty" > cluster-health.json
   docker stats --no-stream > docker-stats.txt
   ```

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

### Quick Contribution Checklist

- [ ] Fork and clone repository
- [ ] Create feature branch
- [ ] Make changes and test locally
- [ ] Pass all CI checks (YAML lint, tests)
- [ ] Update documentation
- [ ] Submit pull request

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- Elastic Stack (Elasticsearch, Logstash, Kibana) by Elastic N.V.
- Docker and Docker Compose by Docker, Inc.
- Senzing by Senzing, Inc.

## Project Status

**Active Development** - This project is actively maintained and welcomes contributions.

**Tested With:**
- Elasticsearch 9.3.0
- Logstash 9.3.0
- Kibana 9.3.0
- Docker 24.0+
- Docker Compose 2.20+

**Last Updated:** 2026-02-04

---

For questions or feedback, please open an issue on GitHub.
