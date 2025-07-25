# Centralized Logging Pipeline in Kubernetes using Fluent Bit, Kafka & OpenSearch

This project sets up a centralized log aggregation pipeline inside a Kubernetes cluster using Fluent Bit, Kafka, and OpenSearch. It was built using Minikube for local development and Helm as the package manager to install each component.

Fluent Bit acts as the log shipper, collecting and parsing logs from Kubernetes pods. Logs are forwarded to Kafka, which buffers and streams the log data to OpenSearch for indexing. OpenSearch Dashboards provides an interface for log exploration and visualization. Grafana is also included for dashboard creation using OpenSearch as a data source.

Custom configurations were defined in `fluent-bit-values.yaml`, `kafka-values.yaml`, and a custom Kafka consumer deployment `kafka-consumer.yaml` to forward messages from Kafka to OpenSearch. Grafana was configured to use OpenSearch as a data source via a `ConfigMap`, allowing visual inspection of logs in real time.

This setup provides a complete log aggregation system with minimal resource consumption, suitable for local development or as a learning tool.