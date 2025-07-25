# Start Minikube with minimal resources
minikube start --memory=4096 --cpus=2 --driver=docker

# Enable metrics server
minikube addons enable metrics-server

# Add all required helm repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add fluent https://fluent.github.io/helm-charts
helm repo add opensearch https://opensearch-project.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create logging namespace
kubectl create namespace logging

# Install OpenSearch with minimal resources
helm install opensearch opensearch/opensearch \
  -n logging \
  --set securityConfig.enabled=true \
  --set securityConfig.admin.password='OMITTED' \
  --set securityConfig.kibanaserver.password='OMITTED' \
  --set replicas=1 \
  --set persistence.enabled=false \
  --set resources.requests.memory=1Gi \
  --set resources.requests.cpu=500m \
  --set extraEnvs[0].name=OPENSEARCH_INITIAL_ADMIN_PASSWORD \
  --set extraEnvs[0].value='OMITTED' \
  --set "extraEnvs[1].name=DISABLE_SECURITY_PLUGIN" \
  --set "extraEnvs[1].value=\"false\"" \
  --set "extraEnvs[2].name=DISABLE_INSTALL_DEMO_CONFIG" \
  --set "extraEnvs[2].value=\"true\""

# Install OpenSearch Dashboards
helm install opensearch-dashboards opensearch/opensearch-dashboards \
  -n logging \
  --set opensearch.username=admin \
  --set opensearch.password='OMITTED' \
  --set opensearchHosts="https://opensearch-cluster-master:9200" \
  --set resources.requests.memory=256Mi \
  --set resources.requests.cpu=100m \
  --set resources.limits.memory=512Mi \
  --set resources.limits.cpu=250m

# Create kafka-values.yaml
vim kafka-values.yaml

# Install Kafka with minimal resources (single node)
helm install kafka bitnami/kafka -n logging -f kafka-values.yaml

# Create fluent-bit-values.yaml
vim fluent-bit-values.yaml

# Install Fluent Bit
helm install fluent-bit fluent/fluent-bit \
  -n logging \
  -f fluent-bit-values.yaml

# Create kafka-consumer.yaml
vim kafka-consumer.yaml

# Apply the consumer
kubectl apply -f kafka-consumer.yaml

# Install Grafana
helm install grafana grafana/grafana -n logging \
  --set persistence.enabled=true \
  --set persistence.size=1Gi \
  --set adminUser=admin \
  --set adminPassword='OMITTED' \
  --set service.type=NodePort \
  --set plugins[0]=grafana-opensearch-datasource \
  --set sidecar.datasources.enabled=true \
  --set sidecar.datasources.label=grafana_datasource \
  --set-string sidecar.datasources.labelValue=1 \
  --set securityContext.runAsUser=472 \
  --set securityContext.runAsGroup=472 \
  --set securityContext.fsGroup=472 \
  --set initChownData.enabled=false

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod -n logging --all --timeout=600s

# Create test workload to generate logs
kubectl create deployment nginx --image=nginx --replicas=2

# Create OpenSearch datasource configuration for Grafana
kubectl apply -n logging -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  labels:
    grafana_datasource: "1"
data:
  opensearch.yaml: |-
    apiVersion: 1
    datasources:
    - name: OpenSearch
      type: grafana-opensearch-datasource
      access: proxy
      url: https://opensearch-cluster-master.logging.svc.cluster.local:9200
      basicAuth: true
      basicAuthUser: admin
      secureJsonData:
        basicAuthPassword: "OMITTED"
      jsonData:
        flavor: Opensearch
        version: 3.1.0
        timeField: "@timestamp"
        tlsSkipVerify: true
        interval: Daily
        logMessageField: log
        logLevelField: level
        maxConcurrentShardRequests: 5
        includeFrozen: false
      isDefault: true
      editable: true
EOF

# Restart Grafana to pick up the new datasource
kubectl rollout restart deployment/grafana -n logging

# Wait for Grafana to restart
kubectl wait --for=condition=ready pod -n logging -l app.kubernetes.io/name=grafana --timeout=300s

# Check all pods status
kubectl get pods -n logging

# Port forward to OpenSearch (run in background)
kubectl port-forward -n logging svc/opensearch-cluster-master 9200:9200 &

# Test OpenSearch connection and check for logs
curl -ku "admin:OMITTED" "https://localhost:9200/_cat/indices?v" || echo "OpenSearch not ready yet"

# Port forward to OpenSearch Dashboards
kubectl port-forward -n logging svc/opensearch-dashboards 5601:5601 &

# Port forward to Grafana
kubectl port-forward -n logging svc/grafana 3000:80 &

# Credentials
Services are now available at:
OpenSearch: https://localhost:9200
OpenSearch Dashboards: http://localhost:5601
Grafana: http://localhost:3000

Login credentials for all services:
Username: admin
Password: OMITTED