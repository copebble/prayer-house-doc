# System Introduction

My server applications run on a cloud server.  
The system is organized into:

- Application development
- Database & Cache
- Batch processing
- Monitoring & Errors alerting
- Logging

## System Architecture

<p align="center"><img width="1384" height="1165" alt="Image" src="https://github.com/user-attachments/assets/71200ee1-a755-4dd7-9357-40a7527ed8a5" /></p>

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
