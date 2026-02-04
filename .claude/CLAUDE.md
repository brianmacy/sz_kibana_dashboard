# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Docker-based ELK (Elasticsearch, Logstash, Kibana) stack deployment for centralized log collection and monitoring of Senzing applications across multiple machines. The stack collects GELF logs from remote Docker containers via UDP and provides pre-configured Kibana dashboards for real-time log analysis.

## Architecture

The deployment consists of four Docker services orchestrated via Docker Compose:

1. **Elasticsearch (9.3.0)**: Single-node cluster for log storage with daily indices (`log-YYYY.MM.dd`)
2. **Logstash (9.3.0)**: GELF input on UDP port 12201, outputs to Elasticsearch
3. **Kibana (9.3.0)**: Web UI on port 5601 for visualization
4. **Dashboard Importer**: One-time setup container that waits for Kibana and imports saved objects

All services run in the `senzing` Docker network and use the `json-file` logging driver (except the importer).

## Common Commands

### Deployment
```bash
# Start all services
docker compose -f kibana.yaml up -d

# Check service status
docker compose -f kibana.yaml ps

# Stop services (preserves data)
docker compose -f kibana.yaml down

# Clean restart (removes all data including volumes)
docker compose -f kibana.yaml down -v
```

### Monitoring
```bash
# View all logs
docker compose -f kibana.yaml logs

# View specific service logs
docker compose -f kibana.yaml logs kibana
docker compose -f kibana.yaml logs logstash
docker compose -f kibana.yaml logs elasticsearch
docker compose -f kibana.yaml logs kibana-dashboard

# Follow logs in real-time
docker compose -f kibana.yaml logs -f

# Check Elasticsearch health
curl http://192.168.2.100:9200/_cluster/health

# Check Kibana status
curl http://192.168.2.100:5601/api/status
```

### Restart Services
```bash
# Restart all services
docker compose -f kibana.yaml restart

# Restart specific service
docker compose -f kibana.yaml restart kibana
docker compose -f kibana.yaml restart logstash
```

## Configuration Details

### Dashboard Structure

The "Senzing Dashboard" includes three saved searches embedded in NDJSON format within `kibana.yaml`:

1. **Error Logs**: Filters messages containing error/fail/exception keywords
   - Query: `message: (*error* OR *Error* OR *ERROR* OR *fail* OR *exception*)`

2. **Workload Statistics**: Filters messages containing "workload"
   - Query: `message: *workload*`

3. **All Recent Logs**: No filter, shows all logs sorted by timestamp descending

All searches use custom column widths:
- `container_name`: 150px
- `source_host`: 120px
- `@timestamp` and `message`: auto-sizing

### Environment Variables

Available overrides in `kibana.yaml`:
- `SENZING_DOCKER_IMAGE_VERSION_ELASTICSEARCH` (default: 9.3.0)
- `SENZING_DOCKER_IMAGE_VERSION_LOGSTASH` (default: 9.3.0)
- `SENZING_DOCKER_IMAGE_VERSION_KIBANA` (default: 9.3.0)
- `SENZING_DOCKER_IMAGE_VERSION_CURLIMAGES_CURL` (default: latest)
- `SENZING_DOCKER_NETWORK` (default: senzing)

### Memory Configuration

Current heap settings:
- Elasticsearch: `-Xmx512m -Xms512m`
- Logstash: `-Xmx256m -Xms256m`

### Network Ports

- **12201/udp**: GELF log ingestion (Logstash)
- **9200/tcp**: Elasticsearch HTTP API
- **9300/tcp**: Elasticsearch transport (node-to-node)
- **9600/tcp**: Logstash monitoring API
- **5601/tcp**: Kibana web UI

### Logstash Pipeline

The Logstash configuration is embedded in `kibana.yaml` as a command-line string:
- Input: GELF on port 12201
- Output: Elasticsearch with daily indices, data streams disabled

### Dashboard Import Process

The `kibana-dashboard` service:
1. Waits for Kibana to respond to `/api/status`
2. Creates temporary NDJSON file with 5 saved objects (index pattern, 3 searches, 1 dashboard)
3. Imports via Kibana's `/api/saved_objects/_import` endpoint
4. Retries until successful (HTTP 200)
5. Exits once import completes

## GitHub Actions CI/CD

The repository includes automated testing via GitHub Actions (`.github/workflows/ci.yml`):

### CI Pipeline Jobs

1. **validate**: YAML linting and Docker Compose validation
   - Uses `yamllint` with configuration from `.yamllint`
   - Validates Docker Compose syntax with `docker compose config`

2. **markdown-lint**: Documentation quality checks
   - Uses `markdownlint-cli2` with configuration from `.markdownlint.json`
   - Lints all `*.md` files in the repository

3. **integration-test**: Full stack integration testing
   - Starts entire ELK stack
   - Verifies Elasticsearch cluster health (green/yellow status)
   - Verifies Kibana availability
   - Verifies Logstash API health
   - Checks dashboard import completion and exit code
   - Verifies "Senzing Dashboard" exists via API
   - Tests GELF log ingestion end-to-end
   - Cleans up with `docker compose down -v`

### Running Tests Locally

```bash
# Lint YAML
pip install yamllint
yamllint -c .yamllint kibana.yaml

# Validate Docker Compose
docker compose -f kibana.yaml config --quiet

# Run full integration test manually
docker compose -f kibana.yaml up -d
# Wait for services...
curl http://localhost:9200/_cluster/health
curl http://localhost:5601/api/status
curl http://localhost:9600
# Send test GELF message
echo '{"version":"1.1","host":"test","short_message":"Test","level":6}' | nc -u localhost 12201
# Cleanup
docker compose -f kibana.yaml down -v
```

## Documentation Structure

The repository includes comprehensive documentation:

- **README.md**: Main entry point with quick start and overview
- **CONFIGURATION.md**: Complete configuration reference with all options
- **SECURITY.md**: Production security hardening guide
- **TROUBLESHOOTING.md**: Detailed problem-solving guide
- **CONTRIBUTING.md**: Guidelines for contributors
- **.claude/CLAUDE.md**: This file (developer guide for Claude Code)

## File Structure

```
.
├── .github/
│   └── workflows/
│       └── ci.yml            # GitHub Actions CI pipeline
├── .claude/
│   └── CLAUDE.md             # This file (developer guide)
├── kibana.yaml               # Docker Compose configuration with embedded dashboard
├── .yamllint                 # YAML linting configuration
├── .markdownlint.json        # Markdown linting configuration
├── .gitignore                # Git ignore patterns
├── README.md                 # User documentation (main entry point)
├── CONFIGURATION.md          # Complete configuration reference
├── SECURITY.md               # Security best practices guide
├── TROUBLESHOOTING.md        # Detailed troubleshooting guide
├── CONTRIBUTING.md           # Contribution guidelines
└── LICENSE                   # Apache 2.0 license
```

## Modifying Dashboard Configuration

To update saved searches or dashboard layout:

1. Make changes via Kibana UI (http://192.168.2.100:5601)
2. Export updated objects:
   ```bash
   curl -X POST "http://192.168.2.100:5601/api/saved_objects/_export" \
     -H "kbn-xsrf: true" \
     -H "Content-Type: application/json" \
     -d '{"type": "dashboard", "includeReferencesDeep": true}'
   ```
3. Update the NDJSON content in `kibana.yaml` lines 81-88
4. Redeploy dashboard importer:
   ```bash
   docker compose -f kibana.yaml restart kibana-dashboard
   ```

## Remote Machine Configuration

Configure Docker containers on remote machines to send logs:

```yaml
services:
  your-app:
    logging:
      driver: gelf
      options:
        gelf-address: "udp://192.168.2.100:12201"
        tag: "{{.Name}}"
```

## Data Persistence

- Elasticsearch data is stored in Docker volume `elasticsearch-data`
- Volume persists through `docker compose down` but is removed with `docker compose down -v`
- Daily indices follow pattern: `log-YYYY.MM.dd`
- Index pattern in Kibana: `log*` with time field `@timestamp`

## Security Notes

- Elasticsearch security is disabled (`xpack.security.enabled: false`)
- Services listen on 0.0.0.0 for multi-machine access
- This configuration is for internal/development networks only
- For production: enable authentication, TLS/SSL, and restrict network access
