# Configuration Reference

Complete configuration reference for the Senzing ELK Stack deployment.

## Table of Contents

- [Environment Variables](#environment-variables)
- [Docker Compose Configuration](#docker-compose-configuration)
- [Elasticsearch Configuration](#elasticsearch-configuration)
- [Logstash Configuration](#logstash-configuration)
- [Kibana Configuration](#kibana-configuration)
- [Dashboard Configuration](#dashboard-configuration)
- [Network Configuration](#network-configuration)
- [Volume Configuration](#volume-configuration)
- [Resource Limits](#resource-limits)
- [Logging Configuration](#logging-configuration)
- [Index Lifecycle Management](#index-lifecycle-management)

## Environment Variables

### Available Environment Variables

All environment variables with defaults:

```bash
# Elasticsearch version
SENZING_DOCKER_IMAGE_VERSION_ELASTICSEARCH=9.3.0

# Logstash version
SENZING_DOCKER_IMAGE_VERSION_LOGSTASH=9.3.0

# Kibana version
SENZING_DOCKER_IMAGE_VERSION_KIBANA=9.3.0

# Curl image version (for dashboard importer)
SENZING_DOCKER_IMAGE_VERSION_CURLIMAGES_CURL=latest

# Docker network name
SENZING_DOCKER_NETWORK=senzing
```

### Using Environment Variables

Create `.env` file in the same directory as `kibana.yaml`:

```bash
# .env
SENZING_DOCKER_IMAGE_VERSION_ELASTICSEARCH=9.3.1
SENZING_DOCKER_IMAGE_VERSION_LOGSTASH=9.3.1
SENZING_DOCKER_IMAGE_VERSION_KIBANA=9.3.1
SENZING_DOCKER_NETWORK=custom-network
```

Or set via command line:

```bash
SENZING_DOCKER_IMAGE_VERSION_ELASTICSEARCH=9.3.1 \
docker compose -f kibana.yaml up -d
```

## Docker Compose Configuration

### Service Dependencies

Dependencies ensure proper startup order:

```
elasticsearch (no dependencies)
  ↓
logstash (depends on elasticsearch)
  ↓
kibana (depends on elasticsearch, logstash)
  ↓
kibana-dashboard (depends on kibana)
```

### Restart Policies

All services use `restart: always` except the dashboard importer (one-time execution).

To change restart policy:

```yaml
services:
  elasticsearch:
    restart: unless-stopped  # Options: no, always, on-failure, unless-stopped
```

### Container Names

Fixed container names for easy reference:

- `senzing-elasticsearch`
- `senzing-logstash`
- `senzing-kibana`
- `senzing-kibana-dashboard`

To customize:

```yaml
services:
  elasticsearch:
    container_name: my-custom-elasticsearch
```

## Elasticsearch Configuration

### Memory Settings

Default heap size: 512MB

```yaml
elasticsearch:
  environment:
    ES_JAVA_OPTS: "-Xmx512m -Xms512m"
```

**Recommendations by system memory:**

- **<2GB RAM**: `-Xmx256m -Xms256m`
- **2-4GB RAM**: `-Xmx512m -Xms512m` (default)
- **4-8GB RAM**: `-Xmx1g -Xms1g`
- **8-16GB RAM**: `-Xmx2g -Xms2g`
- **16GB+ RAM**: `-Xmx4g -Xms4g`

**Rules:**
- Set Xmx and Xms to the same value (prevents heap resizing)
- Don't exceed 50% of system RAM
- Leave room for OS, other services, and Lucene file system cache

### Cluster Configuration

Single-node deployment (default):

```yaml
elasticsearch:
  environment:
    discovery.type: single-node
```

For multi-node cluster:

```yaml
elasticsearch:
  environment:
    cluster.name: senzing-cluster
    node.name: node-1
    discovery.seed_hosts: "elasticsearch-2,elasticsearch-3"
    cluster.initial_master_nodes: "node-1,node-2,node-3"
```

### Index Settings

Configure via Elasticsearch API:

```bash
# Set default number of replicas
curl -X PUT "http://localhost:9200/_template/log-template" \
  -H "Content-Type: application/json" \
  -d '{
    "index_patterns": ["log-*"],
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "refresh_interval": "5s"
    }
  }'
```

### Performance Tuning

```yaml
elasticsearch:
  environment:
    ES_JAVA_OPTS: "-Xmx1g -Xms1g"
    # Increase thread pool size
    thread_pool.write.queue_size: 1000
    # Disable swapping
    bootstrap.memory_lock: 'true'
  ulimits:
    memlock:
      soft: -1
      hard: -1
```

## Logstash Configuration

### Pipeline Configuration

The Logstash pipeline is configured via command-line string in `kibana.yaml`:

```yaml
logstash:
  command:
    - --config.string
    - >-
      input {
        gelf {
          port => 12201
        }
      }
      output {
        elasticsearch {
          hosts => ["http://senzing-elasticsearch:9200"]
          index => "log-%{+YYYY.MM.dd}"
          data_stream => false
        }
      }
```

### Memory Settings

Default heap size: 256MB

```yaml
logstash:
  environment:
    LS_JAVA_OPTS: "-Xmx256m -Xms256m"
```

**Recommendations:**
- **Light workload (<1000 events/sec)**: 256MB (default)
- **Medium workload (1000-5000 events/sec)**: 512MB
- **Heavy workload (>5000 events/sec)**: 1GB+

### Custom Pipeline Configuration

#### Add Filtering

```yaml
logstash:
  command:
    - --config.string
    - >-
      input {
        gelf {
          port => 12201
        }
      }
      filter {
        # Parse JSON in message field
        json {
          source => "message"
          target => "parsed"
        }

        # Add custom field
        mutate {
          add_field => { "environment" => "production" }
        }

        # Drop debug messages
        if [level] == 7 {
          drop {}
        }
      }
      output {
        elasticsearch {
          hosts => ["http://senzing-elasticsearch:9200"]
          index => "log-%{+YYYY.MM.dd}"
          data_stream => false
        }
      }
```

#### Multiple Inputs

```yaml
logstash:
  command:
    - --config.string
    - >-
      input {
        gelf {
          port => 12201
        }
        tcp {
          port => 5000
          codec => json
        }
        beats {
          port => 5044
        }
      }
  ports:
    - 12201:12201/udp
    - 5000:5000
    - 5044:5044
```

#### Conditional Output

```yaml
logstash:
  command:
    - --config.string
    - >-
      input {
        gelf { port => 12201 }
      }
      output {
        # Index errors separately
        if "error" in [message] {
          elasticsearch {
            hosts => ["http://senzing-elasticsearch:9200"]
            index => "errors-%{+YYYY.MM.dd}"
          }
        } else {
          elasticsearch {
            hosts => ["http://senzing-elasticsearch:9200"]
            index => "log-%{+YYYY.MM.dd}"
          }
        }
      }
```

### Pipeline Workers

```yaml
logstash:
  environment:
    PIPELINE_WORKERS: 2  # Number of worker threads
    PIPELINE_BATCH_SIZE: 125  # Events per batch
    PIPELINE_BATCH_DELAY: 50  # Delay in ms
```

## Kibana Configuration

### Elasticsearch Connection

```yaml
kibana:
  environment:
    ELASTICSEARCH_HOSTS: '["http://senzing-elasticsearch:9200"]'
```

For multiple Elasticsearch nodes:

```yaml
kibana:
  environment:
    ELASTICSEARCH_HOSTS: '["http://es-node1:9200","http://es-node2:9200"]'
```

### Server Configuration

```yaml
kibana:
  environment:
    SERVER_NAME: "senzing-kibana"
    SERVER_HOST: "0.0.0.0"
    SERVER_PORT: 5601
    SERVER_BASEPATH: ""  # For reverse proxy
    SERVER_REWRITEBASEPATH: 'false'
```

### Logging Level

```yaml
kibana:
  environment:
    LOGGING_VERBOSE: 'false'  # Set to 'true' for debug logging
```

### Custom Kibana Settings

```yaml
kibana:
  environment:
    ELASTICSEARCH_HOSTS: '["http://senzing-elasticsearch:9200"]'
    KIBANA_DEFAULTAPPID: "dashboard/e37f3b7d-3081-41e4-a9af-01b97c340e14"  # Default to Senzing Dashboard
    TELEMETRY_ENABLED: 'false'
    TELEMETRY_OPTIN: 'false'
```

## Dashboard Configuration

### Dashboard Saved Objects

The dashboard is defined in NDJSON format embedded in `kibana.yaml` (lines 81-88).

### Saved Objects Structure

1. **Index Pattern** (`868351a0-e102-11ec-abf1-5991269c961f`)
   - Pattern: `log*`
   - Time field: `@timestamp`

2. **Error Logs Search** (`1f211439-b386-411b-872a-833126886750`)
   - Query: `message: (*error* OR *Error* OR *ERROR* OR *fail* OR *exception*)`

3. **Workload Statistics Search** (`e2649f60-f5e6-4b0d-a369-e70614d81033`)
   - Query: `message: *workload*`

4. **All Recent Logs Search** (`9b01d74e-a719-4216-abd6-42447c9e7549`)
   - Query: (all logs)

5. **Senzing Dashboard** (`e37f3b7d-3081-41e4-a9af-01b97c340e14`)
   - 2x2 grid layout with saved searches

### Exporting Dashboard Changes

```bash
# Export all saved objects
curl -X POST "http://localhost:5601/api/saved_objects/_export" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "dashboard",
    "includeReferencesDeep": true
  }' > dashboard-export.ndjson

# Export specific dashboard
curl -X POST "http://localhost:5601/api/saved_objects/_export" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "objects": [
      {
        "type": "dashboard",
        "id": "e37f3b7d-3081-41e4-a9af-01b97c340e14"
      }
    ],
    "includeReferencesDeep": true
  }' > senzing-dashboard.ndjson
```

### Importing Dashboards

Update the NDJSON content in `kibana.yaml` between the `cat > /tmp/kibana-objects.ndjson << 'EOF'` and `EOF` markers (lines 81-88).

## Network Configuration

### Docker Network

Default network name: `senzing`

```yaml
networks:
  senzing:
    name: ${SENZING_DOCKER_NETWORK:-senzing}
```

### Custom Network Configuration

```yaml
networks:
  senzing:
    name: senzing
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
          gateway: 172.28.0.1
```

### Port Mappings

**Default ports:**

| Service | Internal Port | External Port | Protocol |
|---------|---------------|---------------|----------|
| Elasticsearch | 9200 | 9200 | TCP |
| Elasticsearch Transport | 9300 | 9300 | TCP |
| Logstash GELF | 12201 | 12201 | UDP |
| Logstash API | 9600 | 9600 | TCP |
| Kibana | 5601 | 5601 | TCP |

**Custom port mapping:**

```yaml
services:
  elasticsearch:
    ports:
      - "9200:9200"  # host:container

  kibana:
    ports:
      - "8080:5601"  # Access Kibana on port 8080
```

## Volume Configuration

### Elasticsearch Data Volume

```yaml
volumes:
  elasticsearch-data:
    driver: local
```

### Custom Volume Location

```yaml
volumes:
  elasticsearch-data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/elasticsearch
```

### NFS Volume

```yaml
volumes:
  elasticsearch-data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.100,rw
      device: ":/mnt/nfs/elasticsearch"
```

### Volume Permissions

```bash
# Create directory with correct permissions
sudo mkdir -p /data/elasticsearch
sudo chown 1000:1000 /data/elasticsearch
sudo chmod 755 /data/elasticsearch
```

## Resource Limits

### CPU Limits

```yaml
services:
  elasticsearch:
    cpus: 2.0  # Limit to 2 CPU cores

  logstash:
    cpus: 1.0

  kibana:
    cpus: 1.0
```

### Memory Limits

```yaml
services:
  elasticsearch:
    mem_limit: 2g
    mem_reservation: 1g

  logstash:
    mem_limit: 512m
    mem_reservation: 256m

  kibana:
    mem_limit: 512m
    mem_reservation: 256m
```

### Complete Resource Configuration

```yaml
services:
  elasticsearch:
    cpus: 2.0
    mem_limit: 2g
    mem_reservation: 1g
    environment:
      ES_JAVA_OPTS: "-Xmx1g -Xms1g"  # Must be < mem_limit
```

## Logging Configuration

### Container Logging

Default: JSON file driver

```yaml
services:
  elasticsearch:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### Available Logging Drivers

```yaml
logging:
  driver: syslog
  options:
    syslog-address: "tcp://192.168.1.100:514"
    tag: "{{.Name}}/{{.ID}}"
```

```yaml
logging:
  driver: gelf
  options:
    gelf-address: "udp://192.168.1.200:12201"
    tag: "elasticsearch"
```

## Index Lifecycle Management

### Configure ILM Policy

```bash
# Create ILM policy for log rotation
curl -X PUT "http://localhost:9200/_ilm/policy/log-policy" \
  -H "Content-Type: application/json" \
  -d '{
    "policy": {
      "phases": {
        "hot": {
          "min_age": "0ms",
          "actions": {
            "rollover": {
              "max_size": "5gb",
              "max_age": "1d"
            }
          }
        },
        "warm": {
          "min_age": "7d",
          "actions": {
            "forcemerge": {
              "max_num_segments": 1
            }
          }
        },
        "delete": {
          "min_age": "30d",
          "actions": {
            "delete": {}
          }
        }
      }
    }
  }'

# Apply policy to index template
curl -X PUT "http://localhost:9200/_index_template/log-template" \
  -H "Content-Type: application/json" \
  -d '{
    "index_patterns": ["log-*"],
    "template": {
      "settings": {
        "index.lifecycle.name": "log-policy",
        "index.lifecycle.rollover_alias": "logs"
      }
    }
  }'
```

### Manual Index Deletion

```bash
# Delete indices older than 30 days
curl -X DELETE "http://localhost:9200/log-$(date -d '30 days ago' +%Y.%m.%d)"

# Delete by pattern
curl -X DELETE "http://localhost:9200/log-2024.01.*"
```

### Automated Cleanup Script

```bash
#!/bin/bash
# cleanup-old-logs.sh

ELASTICSEARCH_URL="http://localhost:9200"
RETENTION_DAYS=30

# Calculate cutoff date
CUTOFF_DATE=$(date -d "$RETENTION_DAYS days ago" +%Y.%m.%d)

# Get list of indices
INDICES=$(curl -s "$ELASTICSEARCH_URL/_cat/indices/log-*?h=index" | sort)

# Delete old indices
echo "$INDICES" | while read index; do
  INDEX_DATE=$(echo $index | grep -oP '\d{4}\.\d{2}\.\d{2}')
  if [[ "$INDEX_DATE" < "$CUTOFF_DATE" ]]; then
    echo "Deleting $index"
    curl -X DELETE "$ELASTICSEARCH_URL/$index"
  fi
done
```

Add to cron:
```bash
# Run daily at 2 AM
0 2 * * * /path/to/cleanup-old-logs.sh >> /var/log/elk-cleanup.log 2>&1
```

## Configuration Examples

### Minimal Configuration (Low Resource)

```yaml
# For systems with 2GB RAM
elasticsearch:
  environment:
    ES_JAVA_OPTS: "-Xmx256m -Xms256m"
  mem_limit: 512m

logstash:
  environment:
    LS_JAVA_OPTS: "-Xmx128m -Xms128m"
  mem_limit: 256m

kibana:
  mem_limit: 256m
```

### Production Configuration (High Resource)

```yaml
# For systems with 16GB+ RAM
elasticsearch:
  environment:
    ES_JAVA_OPTS: "-Xmx4g -Xms4g"
    bootstrap.memory_lock: 'true'
  mem_limit: 8g
  mem_reservation: 4g
  cpus: 4.0
  ulimits:
    memlock:
      soft: -1
      hard: -1

logstash:
  environment:
    LS_JAVA_OPTS: "-Xmx1g -Xms1g"
    PIPELINE_WORKERS: 4
  mem_limit: 2g
  cpus: 2.0

kibana:
  mem_limit: 1g
  cpus: 1.0
```

## Configuration Validation

```bash
# Validate Docker Compose configuration
docker compose -f kibana.yaml config

# Validate with specific environment
SENZING_DOCKER_IMAGE_VERSION_ELASTICSEARCH=9.3.1 \
docker compose -f kibana.yaml config

# Test configuration without starting
docker compose -f kibana.yaml config --services
```

## Additional Resources

- [Elasticsearch Configuration Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/settings.html)
- [Logstash Configuration Documentation](https://www.elastic.co/guide/en/logstash/current/configuration.html)
- [Kibana Configuration Documentation](https://www.elastic.co/guide/en/kibana/current/settings.html)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
