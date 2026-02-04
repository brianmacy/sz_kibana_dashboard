# Security Guide

This guide provides security best practices and configuration steps for hardening the Senzing ELK Stack deployment.

## Table of Contents

- [Security Overview](#security-overview)
- [Current Security Posture](#current-security-posture)
- [Production Security Checklist](#production-security-checklist)
- [Enable Elasticsearch Security](#enable-elasticsearch-security)
- [Configure TLS/SSL](#configure-tlsssl)
- [Network Security](#network-security)
- [Access Control](#access-control)
- [Audit Logging](#audit-logging)
- [Secrets Management](#secrets-management)
- [Security Monitoring](#security-monitoring)
- [Compliance Considerations](#compliance-considerations)
- [Vulnerability Management](#vulnerability-management)

## Security Overview

**WARNING**: The default configuration is designed for internal/development networks and is NOT secure for production or internet-facing deployments.

### Current Security Posture

The default deployment has the following security characteristics:

- ❌ Elasticsearch security disabled (`xpack.security.enabled: false`)
- ❌ No authentication required
- ❌ No encryption (TLS/SSL)
- ❌ Services listen on 0.0.0.0 (all interfaces)
- ❌ No audit logging
- ❌ Default Docker network (bridge mode)
- ❌ No secrets management
- ❌ UDP protocol (GELF) is unencrypted

## Production Security Checklist

Before deploying to production, complete these security tasks:

- [ ] Enable Elasticsearch Security (X-Pack)
- [ ] Configure TLS/SSL for all services
- [ ] Set up authentication (users and roles)
- [ ] Restrict network access (firewall rules)
- [ ] Change default passwords
- [ ] Enable audit logging
- [ ] Implement secrets management
- [ ] Configure index-level security
- [ ] Set up API key authentication
- [ ] Enable security monitoring
- [ ] Regular security updates
- [ ] Backup encryption
- [ ] Compliance validation (if required)

## Enable Elasticsearch Security

### Step 1: Enable X-Pack Security

Update `kibana.yaml`:

```yaml
elasticsearch:
  container_name: senzing-elasticsearch
  environment:
    discovery.type: single-node
    ES_JAVA_OPTS: "-Xmx512m -Xms512m"
    xpack.security.enabled: 'true'  # Changed from 'false'
    xpack.security.transport.ssl.enabled: 'true'
    xpack.security.transport.ssl.verification_mode: certificate
    xpack.security.transport.ssl.keystore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
    xpack.security.transport.ssl.truststore.path: /usr/share/elasticsearch/config/certs/elastic-certificates.p12
```

### Step 2: Generate Certificates

```bash
# Create certificates directory
mkdir -p certs

# Generate CA and node certificates
docker run --rm -v $(pwd)/certs:/certs \
  elasticsearch:9.3.0 \
  bin/elasticsearch-certutil cert \
  --name senzing-elasticsearch \
  --out /certs/elastic-certificates.p12 \
  --pass ""

# Set proper permissions
chmod 644 certs/elastic-certificates.p12
```

### Step 3: Update Docker Compose with Certificates

```yaml
elasticsearch:
  volumes:
    - elasticsearch-data:/usr/share/elasticsearch/data
    - ./certs/elastic-certificates.p12:/usr/share/elasticsearch/config/certs/elastic-certificates.p12:ro
```

### Step 4: Set Initial Passwords

```bash
# Start Elasticsearch
docker compose -f kibana.yaml up -d elasticsearch

# Set built-in user passwords
docker exec -it senzing-elasticsearch bin/elasticsearch-setup-passwords auto

# Save the generated passwords securely!
```

### Step 5: Update Service Configurations

Update Logstash configuration:

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
          hosts => ["https://senzing-elasticsearch:9200"]
          index => "log-%{+YYYY.MM.dd}"
          data_stream => false
          ssl => true
          cacert => "/usr/share/logstash/config/certs/ca.crt"
          user => "elastic"
          password => "${ELASTIC_PASSWORD}"
        }
      }
  environment:
    ELASTIC_PASSWORD: "${ELASTIC_PASSWORD}"
  volumes:
    - ./certs:/usr/share/logstash/config/certs:ro
```

Update Kibana configuration:

```yaml
kibana:
  environment:
    ELASTICSEARCH_HOSTS: '["https://senzing-elasticsearch:9200"]'
    ELASTICSEARCH_USERNAME: "kibana_system"
    ELASTICSEARCH_PASSWORD: "${KIBANA_SYSTEM_PASSWORD}"
    ELASTICSEARCH_SSL_CERTIFICATEAUTHORITIES: "/usr/share/kibana/config/certs/ca.crt"
  volumes:
    - ./certs:/usr/share/kibana/config/certs:ro
```

### Step 6: Create Environment File

Create `.env` file (DO NOT commit to git):

```bash
# .env
ELASTIC_PASSWORD=<your-secure-password>
KIBANA_SYSTEM_PASSWORD=<your-secure-password>
```

Add to `.gitignore`:
```
.env
certs/
```

## Configure TLS/SSL

### Enable HTTPS for Kibana

1. Generate Kibana certificates:

```bash
# Generate Kibana certificate
docker run --rm -v $(pwd)/certs:/certs \
  elasticsearch:9.3.0 \
  bin/elasticsearch-certutil cert \
  --name senzing-kibana \
  --dns senzing-kibana,localhost,192.168.2.100 \
  --out /certs/kibana.p12 \
  --pass ""

# Convert to PEM format
openssl pkcs12 -in certs/kibana.p12 -out certs/kibana.crt -clcerts -nokeys -passin pass:
openssl pkcs12 -in certs/kibana.p12 -out certs/kibana.key -nocerts -nodes -passin pass:
```

2. Update Kibana configuration:

```yaml
kibana:
  environment:
    SERVER_SSL_ENABLED: 'true'
    SERVER_SSL_CERTIFICATE: /usr/share/kibana/config/certs/kibana.crt
    SERVER_SSL_KEY: /usr/share/kibana/config/certs/kibana.key
  ports:
    - 5601:5601
  volumes:
    - ./certs:/usr/share/kibana/config/certs:ro
```

3. Access Kibana via HTTPS:
```
https://192.168.2.100:5601
```

### Enable TLS for GELF Input

Since GELF over UDP doesn't support encryption, use TCP with TLS:

```yaml
logstash:
  command:
    - --config.string
    - >-
      input {
        tcp {
          port => 12201
          ssl_enable => true
          ssl_cert => "/usr/share/logstash/config/certs/logstash.crt"
          ssl_key => "/usr/share/logstash/config/certs/logstash.key"
          ssl_verify => false
          codec => json
        }
      }
  ports:
    - 12201:12201/tcp  # Changed from UDP to TCP
```

Update remote Docker containers:

```yaml
services:
  your-app:
    logging:
      driver: json-file
    # Use a log shipper like Fluent Bit or Filebeat for TLS support
```

## Network Security

### Firewall Configuration

**On ELK Server:**

```bash
# Allow only specific remote machines
sudo iptables -A INPUT -p udp --dport 12201 -s 192.168.2.151 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 12201 -s 192.168.2.142 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 12201 -j DROP

# Restrict Kibana access to admin network
sudo iptables -A INPUT -p tcp --dport 5601 -s 192.168.2.0/24 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 5601 -j DROP

# Block external access to Elasticsearch
sudo iptables -A INPUT -p tcp --dport 9200 -s 127.0.0.1 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 9200 -j DROP

# Save rules
sudo iptables-save > /etc/iptables/rules.v4
```

**Using firewalld (RHEL/CentOS):**

```bash
# Add GELF port for specific sources
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.2.151" port port="12201" protocol="udp" accept'
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.2.142" port port="12201" protocol="udp" accept'

# Add Kibana port for admin subnet
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.2.0/24" port port="5601" protocol="tcp" accept'

# Reload
sudo firewall-cmd --reload
```

### Docker Network Isolation

Use Docker networks to isolate services:

```yaml
networks:
  elk-internal:
    driver: bridge
    internal: true  # No external access
  elk-external:
    driver: bridge

services:
  elasticsearch:
    networks:
      - elk-internal  # Only internal access

  logstash:
    networks:
      - elk-internal
      - elk-external  # Needs external for GELF

  kibana:
    networks:
      - elk-internal
      - elk-external  # Needs external for browser access
```

### VPN Recommendation

For multi-site deployments, use VPN:

- WireGuard
- OpenVPN
- IPsec
- Tailscale (zero-config mesh VPN)

## Access Control

### Create Users and Roles

```bash
# Create read-only user for dashboards
curl -X POST "http://localhost:9200/_security/user/dashboard_viewer" \
  -u elastic:${ELASTIC_PASSWORD} \
  -H "Content-Type: application/json" \
  -d '{
    "password": "secure_password",
    "roles": ["viewer"],
    "full_name": "Dashboard Viewer"
  }'

# Create admin user
curl -X POST "http://localhost:9200/_security/user/admin" \
  -u elastic:${ELASTIC_PASSWORD} \
  -H "Content-Type: application/json" \
  -d '{
    "password": "admin_secure_password",
    "roles": ["superuser"],
    "full_name": "Administrator"
  }'

# Create custom role for log management
curl -X POST "http://localhost:9200/_security/role/log_manager" \
  -u elastic:${ELASTIC_PASSWORD} \
  -H "Content-Type: application/json" \
  -d '{
    "cluster": ["manage_index_templates", "monitor"],
    "indices": [
      {
        "names": ["log-*"],
        "privileges": ["read", "write", "delete", "manage"]
      }
    ]
  }'
```

### API Key Authentication

For programmatic access:

```bash
# Create API key
curl -X POST "http://localhost:9200/_security/api_key" \
  -u elastic:${ELASTIC_PASSWORD} \
  -H "Content-Type: application/json" \
  -d '{
    "name": "logstash-api-key",
    "role_descriptors": {
      "logstash_writer": {
        "cluster": ["manage_index_templates", "monitor"],
        "indices": [
          {
            "names": ["log-*"],
            "privileges": ["write", "create_index", "auto_configure"]
          }
        ]
      }
    }
  }'
```

### Role-Based Access Control (RBAC)

```yaml
# Create roles for different teams
roles:
  operations_team:
    indices:
      - names: ['log-*']
        privileges: ['read', 'view_index_metadata']

  security_team:
    indices:
      - names: ['log-*']
        privileges: ['all']
    cluster: ['monitor']

  developers:
    indices:
      - names: ['log-*app*']
        privileges: ['read']
```

## Audit Logging

### Enable Elasticsearch Audit Logging

```yaml
elasticsearch:
  environment:
    xpack.security.audit.enabled: 'true'
    xpack.security.audit.logfile.events.include: >
      access_denied,
      access_granted,
      authentication_success,
      authentication_failed,
      connection_denied
  volumes:
    - ./audit-logs:/usr/share/elasticsearch/logs
```

### Monitor Audit Logs

```bash
# View audit log
docker exec -it senzing-elasticsearch tail -f /usr/share/elasticsearch/logs/*_audit.json

# Search for failed authentications
docker exec -it senzing-elasticsearch grep "authentication_failed" /usr/share/elasticsearch/logs/*_audit.json
```

## Secrets Management

### Using Docker Secrets (Swarm Mode)

```yaml
# Create secrets
echo "your_elastic_password" | docker secret create elastic_password -

# Use in service
services:
  elasticsearch:
    secrets:
      - elastic_password
    environment:
      ELASTIC_PASSWORD_FILE: /run/secrets/elastic_password

secrets:
  elastic_password:
    external: true
```

### Using Environment Variables with Vault

```bash
# Install Vault
# https://www.vaultproject.io/

# Store secret
vault kv put secret/elk/elastic password="secure_password"

# Retrieve in script
export ELASTIC_PASSWORD=$(vault kv get -field=password secret/elk/elastic)
docker compose -f kibana.yaml up -d
```

### Using External Secrets Operator (Kubernetes)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: elk-credentials
spec:
  secretStoreRef:
    name: vault-backend
  target:
    name: elk-secret
  data:
    - secretKey: elastic-password
      remoteRef:
        key: secret/elk/elastic
        property: password
```

## Security Monitoring

### Monitor Security Events

```bash
# Failed login attempts
curl -s "http://localhost:9200/.security-7/_search" \
  -u elastic:${ELASTIC_PASSWORD} \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "term": {"event.type": "authentication_failed"}
    }
  }'

# Unauthorized access attempts
curl -s "http://localhost:9200/_security/user/_privileges" \
  -u elastic:${ELASTIC_PASSWORD}
```

### Set Up Alerts

Create alerts for security events:

```json
{
  "trigger": {
    "schedule": {"interval": "5m"}
  },
  "input": {
    "search": {
      "request": {
        "indices": [".security-*"],
        "body": {
          "query": {
            "bool": {
              "must": [
                {"term": {"event.type": "authentication_failed"}},
                {"range": {"@timestamp": {"gte": "now-5m"}}}
              ]
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {"ctx.payload.hits.total": {"gt": 5}}
  },
  "actions": {
    "email_admin": {
      "email": {
        "to": "admin@example.com",
        "subject": "Multiple failed login attempts detected"
      }
    }
  }
}
```

## Compliance Considerations

### GDPR Compliance

- Implement data retention policies
- Enable audit logging for all access
- Provide data export/deletion capabilities
- Encrypt data at rest and in transit
- Document data processing activities

### HIPAA Compliance

- Enable audit logging
- Implement access controls
- Encrypt all data (at rest and in transit)
- Regular security assessments
- Business Associate Agreements (BAAs)

### SOC 2 Compliance

- Implement logging and monitoring
- Access control and authentication
- Encryption
- Incident response procedures
- Regular security reviews

## Vulnerability Management

### Regular Updates

```bash
# Check current versions
docker compose -f kibana.yaml images

# Update to latest versions
SENZING_DOCKER_IMAGE_VERSION_ELASTICSEARCH=9.3.1 \
SENZING_DOCKER_IMAGE_VERSION_LOGSTASH=9.3.1 \
SENZING_DOCKER_IMAGE_VERSION_KIBANA=9.3.1 \
docker compose -f kibana.yaml pull

# Recreate containers
docker compose -f kibana.yaml up -d
```

### Security Scanning

```bash
# Scan images for vulnerabilities
docker scan elasticsearch:9.3.0
docker scan logstash:9.3.0
docker scan kibana:9.3.0

# Use Trivy for comprehensive scanning
trivy image elasticsearch:9.3.0
```

### Security Subscriptions

- Subscribe to Elastic Security announcements
- Monitor CVE databases
- Enable automatic security updates (with testing)

## Security Best Practices Summary

1. **Never use default passwords** in production
2. **Always enable TLS/SSL** for all connections
3. **Implement least privilege access** control
4. **Enable audit logging** and monitor regularly
5. **Keep software updated** with security patches
6. **Use secrets management** (Vault, Docker Secrets, etc.)
7. **Network segmentation** and firewall rules
8. **Regular security assessments** and penetration testing
9. **Encrypt backups** and sensitive data
10. **Implement monitoring and alerting** for security events
11. **Document security procedures** and incident response
12. **Regular security training** for team members

## Incident Response

### Security Incident Checklist

1. **Detect**: Monitor logs and alerts
2. **Contain**: Isolate affected systems
3. **Investigate**: Analyze logs and audit trails
4. **Eradicate**: Remove threat and vulnerabilities
5. **Recover**: Restore systems from clean backups
6. **Review**: Post-incident analysis and improvements

### Emergency Contacts

Maintain a security contact list:
- Security team lead
- System administrators
- Legal/compliance team
- Executive sponsor
- Elastic support (if applicable)

## Additional Resources

- [Elastic Security Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-getting-started.html)
- [OWASP Docker Security](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
