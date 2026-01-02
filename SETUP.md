# Elasticsearch Stack - Setup Guide

## 📋 Prerequisites Checklist

Before deploying, ensure you have:

- [ ] Kubernetes cluster (RKE2, K3s, or similar)
- [ ] `kubectl` configured with cluster access
- [ ] Ansible installed with `kubernetes.core` collection
- [ ] Kafka broker accessible from cluster
- [ ] Ingress controller deployed (Traefik, nginx, etc.)
- [ ] DNS entry or hosts file entry for Kibana

## ⚙️ Configuration Steps

### 1. Update Kubeconfig Path

In **all playbook files** (`elasticsearch_create.yml`, `elasticsearch_delete.yml`, `logstash_deploy.yml`):

```yaml
vars:
  kubeconfig: "<< Path to your kubeconfig file >>"
  # Example: "/home/user/.kube/config"
  # Or: "/etc/rancher/rke2/rke2.yaml"
```

### 2. Configure Ingress Domain

In `elasticsearch_create.yml`:

```yaml
vars:
  kibana_url: "<< Your Kibana domain >>"
  # Example: "kibana.example.com"
```

Update your DNS or `/etc/hosts`:
```
<your-ingress-ip> kibana.example.com
```

### 3. Configure Kafka Connection

In `logstash.conf`, update **both input blocks**:

```conf
input {
  kafka {
    bootstrap_servers => "<< Kafka broker IP:port >>"
    # Example: "192.168.1.100:9092"
    topics => ["journals"]
    group_id => "logstash-journals"
    consumer_threads => 1
    decorate_events => true
    codec => json
    tags => ["journals"]
  }

  kafka {
    bootstrap_servers => "<< Kafka broker IP:port >>"
    # Example: "192.168.1.100:9092"
    topics => ["pihole"]
    group_id => "logstash-pihole"
    consumer_threads => 1
    decorate_events => true
    codec => json
    tags => ["pihole"]
  }
}
```

### 4. Verify Kafka Topics

Ensure these topics exist in your Kafka cluster:
- `journals` - for systemd journal logs
- `pihole` - for Pi-hole DNS logs

Create them if needed:
```bash
kafka-topics.sh --create --topic journals --bootstrap-server << Kafka broker IP:port >>
kafka-topics.sh --create --topic pihole --bootstrap-server << Kafka broker IP:port >>
```

## 🚀 Deployment

### Step 1: Deploy Elasticsearch Stack
```bash
ansible-playbook elasticsearch_create.yml
```

This creates:
- Namespace: `elastic-cluster`
- PersistentVolumeClaim (25Gi)
- Elasticsearch deployment
- Kibana deployment
- GeoIP ingest pipeline for DNS IP enrichment
- Ingress for Kibana access

### Step 2: Deploy Logstash
```bash
ansible-playbook logstash_deploy.yml
```

This creates:
- ConfigMap with Logstash pipeline configuration
- Logstash deployment
- Service for monitoring endpoint

### Step 3: Verify Deployment
```bash
# Check all pods are running
kubectl get pods -n elastic-cluster

# Expected output:
# elasticsearch-xxxxxxxxx-xxxxx    1/1     Running
# kibana-xxxxxxxxx-xxxxx           1/1     Running
# logstash-xxxxxxxxx-xxxxx         1/1     Running
```

## 🔍 Access & Verification

### Access Kibana
Visit: `http://<< Your Kibana domain >>`

Or use port-forward:
```bash
kubectl port-forward svc/kibana 5601:5601 -n elastic-cluster
# Visit: http://localhost:5601
```

### Check Data Ingestion

1. **Via kubectl exec**:
```bash
kubectl exec -n elastic-cluster deployment/elasticsearch -- \
  curl -s "http://localhost:9200/_cat/indices?v"
```

You should see indices like:
- `journals-2024.11.26`
- `pihole-2024.11.26`

2. **Via Kibana**:
- Go to "Dev Tools" → Console
- Run: `GET _cat/indices?v`

### Query Sample Logs

**Journald logs**:
```json
GET journals-*/_search
{
  "size": 10,
  "sort": [{"@timestamp": "desc"}]
}
```

**Pihole DNS logs with GeoIP**:
```json
GET pihole-*/_search
{
  "query": {
    "exists": {
      "field": "dns_queried_system_geoip"
    }
  },
  "size": 10
}
```

## 📊 Create Kibana Dashboards

### Create Index Patterns

1. Go to **Stack Management** → **Index Patterns**
2. Create pattern: `journals-*`
   - Time field: `@timestamp`
3. Create pattern: `pihole-*`
   - Time field: `@timestamp`

### Create Visualizations

**1. DNS Query Timeline**
- Visualization: Line chart
- Index pattern: `pihole-*`
- Metrics: Count
- Buckets: Date histogram on `@timestamp`

**2. Top Queried Domains**
- Visualization: Pie chart
- Index pattern: `pihole-*`
- Metrics: Count
- Buckets: Terms aggregation on `dns_queried_system.keyword`
- Size: 10

**3. Blocked vs Allowed**
- Visualization: Metric
- Index pattern: `pihole-*`
- Filter: `dns_query_blocked_response.keyword: exists`

**4. DNS Query Map (with GeoIP)**
- Visualization: Coordinate map
- Index pattern: `pihole-*`
- Geohash: `dns_queried_system_geoip.location`

**5. System Log Levels**
- Visualization: Vertical bar chart
- Index pattern: `journals-*`
- Metrics: Count
- Buckets: Terms on log level field

### Create Dashboard

1. **Dashboards** → **Create Dashboard**
2. Add the visualizations created above
3. Save as "Log Analytics Dashboard"

## 🔧 Customization

### Adjust Resource Limits

In `elasticsearch_create.yml`:
```yaml
resources:
  requests:
    memory: "1Gi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "2000m"
```

In `logstash_deploy.yml`:
```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

### Change Storage Size

In `elasticsearch_create.yml`:
```yaml
spec:
  resources:
    requests:
      storage: 25Gi  # Change this value
```

### Modify Logstash Pipeline

Edit `logstash.conf` to add custom Grok patterns, filters, or outputs.

**Example - Add custom field**:
```conf
filter {
  if "pihole" in [tags] {
    mutate {
      add_field => { "environment" => "production" }
    }
  }
}
```

**Example - Add more Kafka topics**:
```conf
input {
  kafka {
    bootstrap_servers => "<< Kafka broker IP:port >>"
    topics => ["your-new-topic"]
    group_id => "logstash-your-topic"
    consumer_threads => 1
    decorate_events => true
    codec => json
    tags => ["your-topic"]
  }
}

output {
  if "your-topic" in [tags] {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      index => "your-topic-%{+YYYY.MM.dd}"
      document_type => "_doc"
    }
  }
}
```

Then redeploy:
```bash
ansible-playbook logstash_deploy.yml
```

## 🧹 Cleanup

### Delete Everything (keeps PVC)
```bash
ansible-playbook elasticsearch_delete.yml
```

### Delete PVC Manually (if needed)
```bash
kubectl delete pvc elasticsearch-data -n elastic-cluster
```

### Delete Namespace
```bash
kubectl delete namespace elastic-cluster
```

## 🐛 Troubleshooting

### Logstash Not Processing Logs

1. **Check Logstash logs**:
```bash
kubectl logs -f deployment/logstash -n elastic-cluster
```

2. **Verify Kafka connectivity**:
```bash
kubectl exec -n elastic-cluster deployment/logstash -- \
  nc -zv << Kafka broker IP >> 9092
```

3. **Check Logstash monitoring**:
```bash
kubectl port-forward svc/logstash 9600:9600 -n elastic-cluster
curl http://localhost:9600
```

### Elasticsearch Pod CrashLoopBackOff

1. **Check logs**:
```bash
kubectl logs deployment/elasticsearch -n elastic-cluster
```

2. **Common issue - vm.max_map_count**:
The `sysctl` initContainer should handle this, but verify:
```bash
kubectl exec -n elastic-cluster deployment/elasticsearch -- \
  cat /proc/sys/vm/max_map_count
# Should be: 262144
```

3. **Check Java heap settings**:
```bash
kubectl exec -n elastic-cluster deployment/elasticsearch -- \
  env | grep JAVA_OPTS
```

### GeoIP Not Working

1. **Verify pipeline exists**:
```bash
kubectl exec -n elastic-cluster deployment/elasticsearch -- \
  curl "http://localhost:9200/_ingest/pipeline/geoip-pipeline?pretty"
```

2. **Check if template is applied**:
```bash
kubectl exec -n elastic-cluster deployment/elasticsearch -- \
  curl "http://localhost:9200/_index_template/pihole-geoip-template?pretty"
```

3. **Manually test GeoIP**:
```json
POST _ingest/pipeline/geoip-pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "dns_queried_system_ip": "8.8.8.8"
      }
    }
  ]
}
```

### Kibana Can't Connect to Elasticsearch

1. **Check Elasticsearch service**:
```bash
kubectl get svc -n elastic-cluster
```

2. **Test connection from Kibana pod**:
```bash
kubectl exec -n elastic-cluster deployment/kibana -- \
  curl -s http://elasticsearch:9200/_cluster/health
```

## 📊 Monitoring

### View Logstash Monitoring API
```bash
kubectl port-forward svc/logstash 9600:9600 -n elastic-cluster
curl http://localhost:9600
curl http://localhost:9600/_node/stats
```

### Monitor Index Size
```bash
kubectl exec -n elastic-cluster deployment/elasticsearch -- \
  curl "http://localhost:9200/_cat/indices?v&s=store.size:desc"
```

### Check Cluster Health
```bash
kubectl exec -n elastic-cluster deployment/elasticsearch -- \
  curl "http://localhost:9200/_cluster/health?pretty"
```

### Check Ingestion Rate
In Kibana:
- Go to **Stack Monitoring**
- View Elasticsearch metrics
- Check documents indexed per second

## 🔐 Production Hardening

For production deployments, consider:

1. **Enable Elasticsearch Security (X-Pack)**
   - Remove `xpack.security.enabled: "false"`
   - Configure TLS certificates
   - Set up authentication (native, LDAP, SAML)
   - Configure role-based access control (RBAC)

2. **Add Resource Quotas**
   - Set namespace resource limits
   - Configure pod disruption budgets

3. **Implement Backups**
   - Use Elasticsearch snapshot/restore
   - Configure snapshot repository (S3, NFS, etc.)
   - Schedule regular snapshots

4. **Configure TLS**
   - Add cert-manager
   - Enable HTTPS on ingress
   - Use TLS for Elasticsearch inter-node communication

5. **High Availability**
   - Deploy 3+ Elasticsearch nodes
   - Use StatefulSet instead of Deployment
   - Configure replication and sharding

6. **Index Lifecycle Management (ILM)**
   - Define ILM policies for automatic rollover
   - Set retention periods
   - Configure hot-warm-cold architecture

## 🌟 Advanced Features

### GeoIP Enrichment

The stack includes automatic GeoIP enrichment for DNS resolution IPs via ingest pipeline.

**View enriched data**:
```json
GET pihole-*/_search
{
  "query": {
    "exists": {
      "field": "dns_queried_system_geoip.location"
    }
  },
  "fields": [
    "dns_queried_system_ip",
    "dns_queried_system_geoip.country_name",
    "dns_queried_system_geoip.city_name",
    "dns_queried_system_geoip.location"
  ],
  "_source": false
}
```

### Logstash Grok Patterns

Customize Grok patterns in `logstash.conf` for your log format:

```conf
grok {
  match => {
    "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:msg}"
  }
}
```

### Alerting with Kibana

1. Go to **Stack Management** → **Rules and Connectors**
2. Create alert:
   - Condition: `Count > 100` for error logs in 5 minutes
   - Action: Email notification or webhook

### Machine Learning (Requires License)

- Detect anomalies in DNS query patterns
- Identify unusual system log behavior
- Create jobs in **Machine Learning**

## 📚 Next Steps

- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Kibana Documentation](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Logstash Documentation](https://www.elastic.co/guide/en/logstash/current/index.html)
- [Grok Patterns](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html)
- [Index Lifecycle Management](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)
- [Elastic Stack Security](https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html)

## 💡 Tips & Tricks

- **Performance**: Use bulk indexing for better throughput
- **Search Speed**: Create index mapping with proper field types
- **Storage**: Use ILM to manage index retention and reduce storage costs
- **Security**: Always enable X-Pack security in production
- **Backup**: Regular snapshots are essential for disaster recovery
- **Monitoring**: Use Stack Monitoring to track cluster health































