# Troubleshooting Guide

This guide provides solutions to common issues encountered when deploying and operating the Senzing ELK Stack.

## Table of Contents

- [Quick Diagnostics](#quick-diagnostics)
- [Services Not Starting](#services-not-starting)
- [No Logs Appearing](#no-logs-appearing)
- [Dashboard Issues](#dashboard-issues)
- [Performance Problems](#performance-problems)
- [Network Connectivity](#network-connectivity)
- [Disk Space Issues](#disk-space-issues)
- [Memory Issues](#memory-issues)
- [Getting Help](#getting-help)

## Quick Diagnostics

Run these commands first to gather basic system information:

```bash
# Check service status
docker compose -f kibana.yaml ps

# Check Elasticsearch health
curl -s http://localhost:9200/_cluster/health?pretty

# Check Kibana status
curl -s http://localhost:5601/api/status | jq

# Check Logstash API
curl -s http://localhost:9600/?pretty

# View recent logs
docker compose -f kibana.yaml logs --tail=50

# Check disk space
df -h

# Check memory usage
free -h
docker stats --no-stream
```

## Services Not Starting

### Elasticsearch Won't Start

**Symptoms:**
- Container exits immediately or enters restart loop
- Error: "max virtual memory areas vm.max_map_count [65530] is too low"

**Solutions:**

1. Increase vm.max_map_count (Linux/macOS):
   ```bash
   # Temporary (until reboot)
   sudo sysctl -w vm.max_map_count=262144

   # Permanent
   echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
   sudo sysctl -p
   ```

2. Check available disk space:
   ```bash
   df -h
   # Elasticsearch needs at least 10GB free
   ```

3. Verify permissions on data volume:
   ```bash
   docker volume inspect elasticsearch-data
   ```

4. Check Elasticsearch logs:
   ```bash
   docker compose -f kibana.yaml logs elasticsearch
   ```

### Logstash Won't Start

**Symptoms:**
- Configuration syntax errors
- Cannot connect to Elasticsearch
- Port 12201 already in use

**Solutions:**

1. Validate configuration:
   ```bash
   docker compose -f kibana.yaml config
   ```

2. Check port availability:
   ```bash
   # Check if port 12201 is already in use
   sudo lsof -i :12201
   # Or on Linux
   sudo netstat -tulpn | grep 12201
   ```

3. Verify Elasticsearch is running:
   ```bash
   curl http://localhost:9200
   ```

4. Check Logstash logs:
   ```bash
   docker compose -f kibana.yaml logs logstash
   ```

### Kibana Won't Start

**Symptoms:**
- Cannot connect to Elasticsearch
- Timeout waiting for Elasticsearch
- Port 5601 already in use

**Solutions:**

1. Verify Elasticsearch is healthy:
   ```bash
   curl http://localhost:9200/_cluster/health
   ```

2. Check port availability:
   ```bash
   sudo lsof -i :5601
   ```

3. Increase Kibana startup timeout:
   ```yaml
   # Add to kibana service in kibana.yaml
   environment:
     ELASTICSEARCH_HOSTS: '["http://senzing-elasticsearch:9200"]'
     ELASTICSEARCH_REQUESTTIMEOUT: 90000
   ```

4. Check Kibana logs:
   ```bash
   docker compose -f kibana.yaml logs kibana
   ```

### Dashboard Importer Fails

**Symptoms:**
- Exit code not 0
- Dashboard not visible in Kibana
- Import errors in logs

**Solutions:**

1. Check importer logs:
   ```bash
   docker compose -f kibana.yaml logs kibana-dashboard
   ```

2. Verify Kibana is ready:
   ```bash
   curl http://localhost:5601/api/status
   ```

3. Manually re-run the import:
   ```bash
   docker compose -f kibana.yaml restart kibana-dashboard
   ```

4. Verify index pattern exists:
   ```bash
   curl -s "http://localhost:5601/api/saved_objects/index-pattern/868351a0-e102-11ec-abf1-5991269c961f" | jq
   ```

## No Logs Appearing

### Remote Machine Configuration Issues

**Symptoms:**
- No logs visible in Kibana
- GELF messages not reaching Logstash
- Empty indices

**Solutions:**

1. Test GELF connectivity from remote machine:
   ```bash
   # From remote machine
   echo '{"version":"1.1","host":"test","short_message":"Test message","level":6}' | \
     nc -u 192.168.2.100 12201
   ```

2. Verify message received in Logstash:
   ```bash
   docker compose -f kibana.yaml logs logstash | grep -i "test message"
   ```

3. Check firewall rules:
   ```bash
   # On ELK server
   sudo iptables -L -n | grep 12201
   # Or
   sudo firewall-cmd --list-ports
   ```

4. Verify Docker container GELF configuration:
   ```bash
   # On remote machine
   docker inspect <container-name> | grep -A5 LogConfig
   ```

5. Check if logs are in Elasticsearch:
   ```bash
   curl -s "http://localhost:9200/log-*/_search?size=10&sort=@timestamp:desc&pretty"
   ```

### Index Pattern Not Found

**Symptoms:**
- "No indices match pattern log*"
- Dashboard shows "No results found"

**Solutions:**

1. Verify indices exist:
   ```bash
   curl http://localhost:9200/_cat/indices?v
   ```

2. Check index pattern in Kibana:
   ```bash
   curl -s "http://localhost:5601/api/saved_objects/index-pattern/868351a0-e102-11ec-abf1-5991269c961f" | jq
   ```

3. Manually create index pattern via Kibana UI:
   - Go to Stack Management → Index Patterns
   - Create index pattern: `log*`
   - Select time field: `@timestamp`

4. Send test log to create initial index:
   ```bash
   echo '{"version":"1.1","host":"test","short_message":"Initial test","level":6,"_container_name":"test"}' | \
     nc -u localhost 12201
   sleep 5
   curl http://localhost:9200/_cat/indices?v
   ```

## Dashboard Issues

### Dashboard Not Loading

**Symptoms:**
- 404 error on dashboard
- Empty dashboard page
- Missing visualizations

**Solutions:**

1. Verify dashboard exists:
   ```bash
   curl -s "http://localhost:5601/api/saved_objects/dashboard/e37f3b7d-3081-41e4-a9af-01b97c340e14" | jq
   ```

2. Check saved searches exist:
   ```bash
   # Error Logs search
   curl -s "http://localhost:5601/api/saved_objects/search/1f211439-b386-411b-872a-833126886750" | jq

   # Workload Statistics search
   curl -s "http://localhost:5601/api/saved_objects/search/e2649f60-f5e6-4b0d-a369-e70614d81033" | jq

   # All Recent Logs search
   curl -s "http://localhost:5601/api/saved_objects/search/9b01d74e-a719-4216-abd6-42447c9e7549" | jq
   ```

3. Re-import dashboard:
   ```bash
   docker compose -f kibana.yaml restart kibana-dashboard
   ```

4. Check for conflicts:
   - Go to Kibana → Stack Management → Saved Objects
   - Look for duplicate or conflicting objects

### Saved Searches Show No Data

**Symptoms:**
- Dashboard loads but panels are empty
- "No results found" in search panels

**Solutions:**

1. Check time range:
   - Adjust time picker in Kibana (top-right)
   - Try "Last 24 hours" or "Last 7 days"

2. Verify data exists for the search query:
   ```bash
   # Test error logs query
   curl -s "http://localhost:9200/log-*/_search?q=message:*error*&size=1" | jq .hits.total
   ```

3. Refresh field list:
   - Go to Stack Management → Index Patterns → log*
   - Click refresh button

## Performance Problems

### Slow Query Performance

**Symptoms:**
- Dashboard takes long to load
- Search timeouts
- High CPU usage

**Solutions:**

1. Check Elasticsearch cluster health:
   ```bash
   curl -s "http://localhost:9200/_cluster/health?pretty"
   ```

2. Reduce time range being queried

3. Check for unassigned shards:
   ```bash
   curl -s "http://localhost:9200/_cat/shards?v" | grep UNASSIGNED
   ```

4. Monitor slow queries:
   ```bash
   curl -s "http://localhost:9200/_nodes/stats/indices/search?pretty"
   ```

5. Increase heap memory (if resources available):
   ```yaml
   # In kibana.yaml, update Elasticsearch service
   environment:
     ES_JAVA_OPTS: "-Xmx1g -Xms1g"  # Increase from 512m
   ```

### High Memory Usage

**Symptoms:**
- Out of memory errors
- Container killed by OOM killer
- System becomes unresponsive

**Solutions:**

1. Check current memory usage:
   ```bash
   docker stats
   ```

2. Reduce heap sizes if system has limited memory:
   ```yaml
   # For systems with <4GB RAM
   elasticsearch:
     environment:
       ES_JAVA_OPTS: "-Xmx256m -Xms256m"

   logstash:
     environment:
       LS_JAVA_OPTS: "-Xmx128m -Xms128m"
   ```

3. Implement index lifecycle management:
   ```bash
   # Delete old indices
   curl -X DELETE "http://localhost:9200/log-2024.01.*"
   ```

4. Monitor Java heap usage:
   ```bash
   curl -s "http://localhost:9200/_nodes/stats/jvm?pretty"
   ```

## Network Connectivity

### Cannot Access Kibana UI

**Symptoms:**
- Connection refused on port 5601
- Timeout connecting to Kibana

**Solutions:**

1. Verify Kibana is running:
   ```bash
   docker compose -f kibana.yaml ps kibana
   ```

2. Check if port is listening:
   ```bash
   netstat -tulpn | grep 5601
   ```

3. Test from localhost first:
   ```bash
   curl http://localhost:5601
   ```

4. Check firewall rules:
   ```bash
   sudo iptables -L -n | grep 5601
   ```

5. Verify Docker port mapping:
   ```bash
   docker port senzing-kibana
   ```

### GELF Messages Not Reaching Logstash

**Symptoms:**
- Remote containers logging to GELF but logs not appearing
- No errors in container logs

**Solutions:**

1. Test UDP connectivity:
   ```bash
   # From remote machine
   nc -u -v -z 192.168.2.100 12201
   ```

2. Capture packets on ELK server:
   ```bash
   sudo tcpdump -i any udp port 12201 -n
   ```

3. Check Logstash input statistics:
   ```bash
   curl -s "http://localhost:9600/_node/stats/pipelines?pretty" | jq '.pipelines.main.plugins.inputs'
   ```

4. Verify no NAT/routing issues:
   ```bash
   traceroute 192.168.2.100
   ```

## Disk Space Issues

### Elasticsearch Disk Full

**Symptoms:**
- Error: "flood stage disk watermark exceeded"
- Indices become read-only
- Cannot index new documents

**Solutions:**

1. Check disk usage:
   ```bash
   df -h
   curl -s "http://localhost:9200/_cat/allocation?v"
   ```

2. Delete old indices:
   ```bash
   # List indices by size
   curl -s "http://localhost:9200/_cat/indices?v&s=store.size:desc"

   # Delete old indices
   curl -X DELETE "http://localhost:9200/log-2024.01.*"
   ```

3. Reset read-only index block:
   ```bash
   curl -X PUT "http://localhost:9200/_all/_settings" \
     -H 'Content-Type: application/json' \
     -d '{"index.blocks.read_only_allow_delete": null}'
   ```

4. Configure disk watermarks:
   ```bash
   curl -X PUT "http://localhost:9200/_cluster/settings" \
     -H 'Content-Type: application/json' \
     -d '{
       "transient": {
         "cluster.routing.allocation.disk.watermark.low": "85%",
         "cluster.routing.allocation.disk.watermark.high": "90%",
         "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
       }
     }'
   ```

### Docker Volume Full

**Symptoms:**
- Cannot write to volume
- Container fails to start

**Solutions:**

1. Check volume size:
   ```bash
   docker system df -v
   ```

2. Prune unused Docker data:
   ```bash
   docker system prune -a --volumes
   ```

3. Move data volume to larger disk (requires backup/restore)

## Memory Issues

### Java Heap Space Errors

**Symptoms:**
- OutOfMemoryError in logs
- Services crash randomly
- Slow performance

**Solutions:**

1. Check heap usage:
   ```bash
   # Elasticsearch
   curl -s "http://localhost:9200/_nodes/stats/jvm?pretty" | jq '.nodes[].jvm.mem'

   # Logstash
   curl -s "http://localhost:9600/_node/stats/jvm?pretty" | jq '.jvm.mem'
   ```

2. Increase heap size (ensure host has enough RAM):
   ```yaml
   elasticsearch:
     environment:
       ES_JAVA_OPTS: "-Xmx1g -Xms1g"

   logstash:
     environment:
       LS_JAVA_OPTS: "-Xmx512m -Xms512m"
   ```

3. Reduce number of indices/data volume

4. Enable heap dumps for analysis:
   ```yaml
   environment:
     ES_JAVA_OPTS: "-Xmx512m -Xms512m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp"
   ```

## Getting Help

### Collecting Debug Information

Before requesting help, collect this information:

```bash
# System info
uname -a
docker version
docker compose version

# Service status
docker compose -f kibana.yaml ps

# Service logs (last 100 lines)
docker compose -f kibana.yaml logs --tail=100 > elk-logs.txt

# Cluster health
curl -s "http://localhost:9200/_cluster/health?pretty" > cluster-health.json
curl -s "http://localhost:9200/_cat/indices?v" > indices.txt
curl -s "http://localhost:9200/_cat/nodes?v" > nodes.txt

# Resource usage
docker stats --no-stream > docker-stats.txt
df -h > disk-usage.txt
free -h > memory-usage.txt

# Network connectivity
netstat -tulpn > ports.txt
```

### Support Channels

- **GitHub Issues**: Report bugs or request features at the repository
- **Documentation**: Check README.md, CONFIGURATION.md, and SECURITY.md
- **Elasticsearch Forums**: https://discuss.elastic.co
- **Docker Forums**: https://forums.docker.com

### Debug Mode

Enable debug logging for detailed troubleshooting:

```yaml
# Add to kibana.yaml
elasticsearch:
  environment:
    ES_JAVA_OPTS: "-Xmx512m -Xms512m"
    logger.level: DEBUG

logstash:
  environment:
    LOG_LEVEL: debug

kibana:
  environment:
    LOGGING_VERBOSE: true
```

Then restart services:
```bash
docker compose -f kibana.yaml down
docker compose -f kibana.yaml up -d
```
