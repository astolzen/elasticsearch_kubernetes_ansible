# Elasticsearch DNS Analytics Stack

Complete DNS log analytics solution using Elasticsearch, Kibana, Logstash, and Kafka for real-time DNS query analysis and geolocation visualization.
_AI Tools like Claude, Gemini, Ollama, qwen3-coder, gpt-oss, Continue and Cursor AI assisted in the generation of this Code and Documentation_

## Overview

This project provides automated deployment of a comprehensive DNS analytics platform using the Elastic Stack that:
- Collects DNS logs from Pi-hole via Filebeat
- Streams data through Kafka for scalability
- Processes logs with Logstash
- Enriches DNS resolution data with GeoIP information
- Visualizes DNS traffic on interactive maps in Kibana

## Architecture

```
┌─────────────┐      ┌─────────┐      ┌──────────┐      ┌────────────────┐      ┌─────────┐
│  Filebeat   │─────>│  Kafka  │─────>│ Logstash │─────>│ Elasticsearch  │─────>│ Kibana  │
│  (Pi-hole)  │      │ Topics  │      │ Pipeline │      │    Indices     │      │  Maps   │
└─────────────┘      └─────────┘      └──────────┘      └────────────────┘      └─────────┘
  - journals            - journals      - JSON parse       - journals-*           - DNS queries
  - pihole logs         - pihole        - Grok patterns    - pihole-*             - Blocked domains
                                       - GeoIP enrich                             - Geographic viz
```

## Components

### 1. Elasticsearch 9.2.1
- Single-node deployment
- No security (for lab/internal use)
- Persistent storage (25Gi PVC)
- GeoIP ingest pipeline for DNS resolution data

### 2. Kibana 9.2.1
- Web UI for visualization
- Ingress configured: `<< Your Kibana domain >>`
- Index patterns for journals and pihole logs

### 3. Logstash 9.2.1
- Kafka input plugin (<< Kafka broker IP:port >>)
- Processing pipeline:
  - **journals**: systemd journal logs
  - **pihole**: Pi-hole DNS logs
- JSON parsing and field extraction
- Grok pattern matching for DNS queries

### 4. Data Sources
- **Kafka Broker**: << Kafka broker IP:port >> (no auth, no TLS)
- **Topics**:
  - `journals`: Systemd journal logs from filebeat
  - `pihole`: Pi-hole DNS logs from filebeat

## Prerequisites

- Kubernetes cluster with kubectl access
- Ansible installed locally
- Kubeconfig file: `<< Path to your kubeconfig file >>`
- Kafka broker running at << Kafka broker IP:port >>
- Filebeat instances shipping logs to Kafka

## Files

```
kubernetes/elasticsearch/
├── README.md                    # This file
├── elasticsearch_create.yml     # Deploy Elasticsearch + Kibana + GeoIP
├── elasticsearch_delete.yml     # Remove deployment (preserve PV/PVC)
├── logstash.conf                # Logstash pipeline configuration
└── logstash_deploy.yml          # Deploy Logstash
```

## Deployment

### Initial Setup

1. **Deploy Elasticsearch and Kibana:**
   ```bash
   ansible-playbook kubernetes/elasticsearch/elasticsearch_create.yml
   ```

   This playbook:
   - Creates namespace `elastic-cluster`
   - Deploys Elasticsearch 9.2.1
   - Deploys Kibana 9.2.1
   - Creates GeoIP ingest pipeline
   - Configures index template for automatic GeoIP enrichment
   - Creates ingress for Kibana access

2. **Deploy Logstash:**
   ```bash
   ansible-playbook kubernetes/elasticsearch/logstash_deploy.yml
   ```

   This playbook:
   - Reads pipeline configuration from `logstash.conf`
   - Creates ConfigMap with pipeline
   - Deploys Logstash with Kafka integration
   - Waits for pod to be ready

### Clean Removal

To remove Elasticsearch while preserving data:
```bash
ansible-playbook kubernetes/elasticsearch/elasticsearch_delete.yml
```

**Note:** This preserves the PVC `elasticsearch-data` so you don't lose your data.

## Configuration

### Elasticsearch Variables
Located in `elasticsearch_create.yml`:
```yaml
kubeconfig: "<< Path to your kubeconfig file >>"
elastic_ns: "elastic-cluster"
cluster_name: "elastic-single"
elasticsearch_version: "9.2.1"
kibana_version: "9.2.1"
```

### Logstash Pipeline
Located in `logstash.conf`:

#### Input Section
- Kafka consumer for `journals` and `pihole` topics
- JSON codec for automatic parsing
- Consumer groups: `logstash-journals` and `logstash-pihole`

#### Filter Section
- **JSON parsing**: Automatic from Kafka codec
- **Grok patterns** for:
  - DNS queries: `query[A]` and `query[AAAA]`
  - Blocked domains: `gravity blocked`
  - DNS resolutions: `reply`, `cached`, `cached-stale`
- **Pipeline tagging**: Identifies source topic

#### Output Section
- **journals**: → `journals-YYYY.MM.DD` index
- **pihole**: → `pihole-YYYY.MM.DD` index (with GeoIP pipeline)

## Data Fields Extracted

### DNS Query Fields
From messages like: `query[A] www.microsoft.com from << Client IP address >>`
- `dns_queried_system`: Domain being queried
- `dns_query_from`: IP address making the query

### Blocked Query Fields
From messages like: `gravity blocked incoming.telemetry.mozilla.org is 0.0.0.0`
- `dns_queried_system`: Blocked domain
- `dns_query_blocked_response`: Block response

### DNS Resolution Fields
From messages like: `reply slackb.com is 34.202.253.140`
- `dns_queried_system`: Domain name
- `dns_queried_system_ip`: Resolved IP address
- `dns_queried_system_geoip`: GeoIP information (added by Elasticsearch)
  - `continent_name`
  - `country_name`
  - `country_iso_code`
  - `region_name`
  - `city_name`
  - `location`: Coordinates in geo_point format

## GeoIP Enrichment

The Elasticsearch ingest pipeline automatically enriches DNS resolution IPs with geographic data:

1. **Pipeline**: `geoip-pipeline`
   - Extracts location from `dns_queried_system_ip`
   - Adds geographic metadata
   - Converts location to geo_point format for mapping

2. **Index Template**: `pihole-geoip-template`
   - Applies to all `pihole-*` indices
   - Automatically uses `geoip-pipeline`
   - Ensures `location` field is mapped as `geo_point`

## Access Information

### Elasticsearch API
```bash
kubectl port-forward svc/elasticsearch 9200:9200 -n elastic-cluster
# Then access: http://localhost:9200
```

### Kibana
- **URL**: http://<< Your Kibana domain >>
- **Port-forward**: 
  ```bash
  kubectl port-forward svc/kibana 5601:5601 -n elastic-cluster
  # Then access: http://localhost:5601
  ```

### Logstash Monitoring API
```bash
kubectl port-forward svc/logstash 9600:9600 -n elastic-cluster
# Then access: http://localhost:9600
```

## Creating Visualizations

### DNS Query Map (Geographic Distribution)

1. **Create Index Pattern** (if not exists):
   - Go to **Stack Management** → **Index Patterns**
   - Create pattern: `pihole-*`
   - Time field: `@timestamp`

2. **Create Coordinate Map**:
   - **Visualize** → **Create visualization** → **Coordinate Map**
   - Select index pattern: `pihole-*`
   - **Buckets** → **Add** → **Geo coordinates**
   - **Aggregation**: Geohash
   - **Field**: `dns_queried_system_geoip.location`
   - Click **Update**

3. **Alternative: Use Maps App**:
   - **Visualize** → **Create visualization** → **Maps**
   - Add layer → **Documents**
   - Index pattern: `pihole-*`
   - Geospatial field: `dns_queried_system_geoip.location`

### Sample Queries

**All DNS queries with source IP:**
```
dns_query_from: *
```

**All blocked domains:**
```
dns_query_blocked_response: *
```

**DNS resolutions with geolocation:**
```
_exists_:dns_queried_system_geoip.location
```

**Queries from specific IP:**
```
dns_query_from: "10.0.1.100"
```

**Domains resolved to specific country:**
```
dns_queried_system_geoip.country_name: "United States"
```

## Monitoring

### Check Logstash Status
```bash
kubectl logs -f deployment/logstash -n elastic-cluster
```

### Check Elasticsearch Indices
```bash
kubectl exec -n elastic-cluster deployment/elasticsearch -- \
  curl -s http://localhost:9200/_cat/indices?v
```

### Check Document Count
```bash
# Journals
kubectl exec -n elastic-cluster deployment/elasticsearch -- \
  curl -s http://localhost:9200/journals-*/_count

# Pihole
kubectl exec -n elastic-cluster deployment/elasticsearch -- \
  curl -s http://localhost:9200/pihole-*/_count
```

### Verify GeoIP Enrichment
```bash
kubectl exec -n elastic-cluster deployment/elasticsearch -- \
  curl -s -X GET "http://localhost:9200/pihole-*/_search?size=1" \
  -H 'Content-Type: application/json' \
  -d '{"query":{"exists":{"field":"dns_queried_system_geoip.location"}}}'
```

### Check Logstash Pipeline Stats
```bash
kubectl exec -n elastic-cluster deployment/logstash -- \
  curl -s http://localhost:9600/_node/stats/pipelines
```

## Troubleshooting

### Logstash Pod Crashing
1. Check logs: `kubectl logs deployment/logstash -n elastic-cluster`
2. Verify Kafka connectivity: Ensure << Kafka broker IP:port >> is reachable
3. Check ConfigMap: `kubectl get configmap logstash-config -n elastic-cluster -o yaml`
4. Validate logstash.conf syntax

### No GeoIP Data in Visualizations
1. Verify pipeline exists:
   ```bash
   kubectl exec -n elastic-cluster deployment/elasticsearch -- \
     curl -s http://localhost:9200/_ingest/pipeline/geoip-pipeline
   ```
2. Check index template:
   ```bash
   kubectl exec -n elastic-cluster deployment/elasticsearch -- \
     curl -s http://localhost:9200/_index_template/pihole-geoip-template
   ```
3. Recreate index pattern in Kibana

### Kibana Not Loading
1. Check pod status: `kubectl get pods -n elastic-cluster`
2. Check logs: `kubectl logs deployment/kibana -n elastic-cluster`
3. Verify Elasticsearch connection in Kibana logs
4. Verify ingress: `kubectl get ingress -n elastic-cluster`

### Kafka Connection Issues
1. Test connectivity from Logstash pod:
   ```bash
   kubectl exec -n elastic-cluster deployment/logstash -- \
     nc -zv << Kafka broker IP >> 9092
   ```
2. Check Logstash logs for Kafka errors
3. Verify Kafka topic exists and has data

## Key Differences from OpenSearch Stack

| Component | OpenSearch Stack | Elasticsearch Stack |
|-----------|------------------|---------------------|
| Search Engine | OpenSearch 3.3.2 | Elasticsearch 9.2.1 |
| UI | OpenSearch Dashboards 3.3.0 | Kibana 9.2.1 |
| Processing | Data Prepper 2.12.2 | Logstash 9.2.1 |
| Config Format | YAML pipelines | Logstash DSL (Ruby-based) |
| Image Source | opensearchproject/* | docker.elastic.co/* |
| Namespace | opensearch-cluster | elastic-cluster |
| Ingress | oss.<< Your domain name >> | << Your Kibana domain >> |

## Storage

- **PVC Name**: `elasticsearch-data`
- **Size**: 25Gi
- **Access Mode**: ReadWriteOnce
- **Location**: `/usr/share/elasticsearch/data` in Elasticsearch pod

**Note**: The delete playbook preserves this PVC to prevent data loss.

## Security Notes

⚠️ **This deployment disables security for simplicity:**
- No authentication required for Elasticsearch
- No TLS/SSL encryption
- Suitable for internal/lab environments only

**For production use, enable:**
- X-Pack Security
- TLS/SSL certificates
- Authentication (basic auth, LDAP, SAML)
- Network policies
- RBAC

## Performance Tuning

### Increase Logstash Resources
Edit `logstash_deploy.yml`:
```yaml
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "2000m"
```

### Adjust Logstash Workers
Edit `logstash.conf`, add to filter section:
```ruby
pipeline.workers: 4
pipeline.batch.size: 125
```

### Elasticsearch JVM Settings
Edit `elasticsearch_create.yml`, modify ES_JAVA_OPTS:
```yaml
- name: ES_JAVA_OPTS
  value: "-Xms1g -Xmx1g"
```

## Backup and Recovery

### Backup PVC
```bash
kubectl exec -n elastic-cluster deployment/elasticsearch -- \
  tar czf /tmp/elasticsearch-backup.tar.gz /usr/share/elasticsearch/data

kubectl cp elastic-cluster/elasticsearch-xxxxx:/tmp/elasticsearch-backup.tar.gz \
  ./elasticsearch-backup.tar.gz
```

### Restore
```bash
kubectl cp ./elasticsearch-backup.tar.gz \
  elastic-cluster/elasticsearch-xxxxx:/tmp/

kubectl exec -n elastic-cluster deployment/elasticsearch -- \
  tar xzf /tmp/elasticsearch-backup.tar.gz -C /
```

## Version History

- **Initial Release**: Elasticsearch 9.2.1, Kibana 9.2.1, Logstash 9.2.1
- GeoIP enrichment with automatic geo_point mapping
- Kafka integration with journals and pihole topics
- DNS query analytics with geolocation visualization

## Contributing

To modify this deployment:
1. Update configuration in YAML or .conf files
2. Test changes in dev environment
3. Run playbooks to apply changes
4. Verify with monitoring commands

## Support

For issues or questions:
- Check logs: `kubectl logs -n elastic-cluster <pod-name>`
- Review Elasticsearch documentation: https://www.elastic.co/guide/
- Check Logstash docs: https://www.elastic.co/guide/en/logstash/

## License

Internal use - Adjust according to your organization's requirements.


