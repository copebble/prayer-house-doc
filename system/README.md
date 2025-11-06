# System Introduction

My server applications run on a cloud server. 
The system is organized into:

- Application development
- Database & Cache
- Batch processing
- Monitoring & Errors alerting
- Logging

## System Architecture

<p align="center"><img width="1384" height="1162" alt="Image" src="https://github.com/user-attachments/assets/1731ab14-653b-483c-b7c9-657e95db3764" /></p>

- Application
  - Java, Kotlin
  - Spring Boot
  - k8s
- Database & Event
  - MySQL
  - Redis (cache)
  - Kafka
- Batch scheduling
  - Airflow
- Monitoring & Errors alerting
  - Prometheus, Alertmanager, Grafana
  - Sentry, GlitchTip, Slack
- Logging
  - EFK stack (Elasticsearch, Fluentd, Kibana)
  - Elastic APM
- Secret
  - Hashicorp Vault
