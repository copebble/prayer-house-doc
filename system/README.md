# System Introduction

My server applications run on a cloud server.  
The system is organized into:

- Application development
- Database & Cache
- Batch processing
- Monitoring & Errors alerting
- Logging

## System Architecture

<p align="center"><img width="1384" height="1165" alt="Image" src="https://github.com/user-attachments/assets/6d682872-eaf1-44ab-b9cf-ff0e2c827257" /></p>

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

## Cluster Structure

<p align="center"><img width="1004" height="1137" alt="Image" src="https://github.com/user-attachments/assets/14f74131-b181-4467-9240-a19c781a0149" /></p>

1. **Gateway Node Cluster**

This cluster is responsible for handling all external traffic entering the system.
- Traffic routing
- External to internal network isolation: preventing direct access to internal API server
- Rate limiting: controlling request volume

2. **Production Node Cluster**

This cluster manages all live production instances running project applications.
It runs critical business processes that may cause service disruption for real users.
Therefore, I must prioritize stability and reliability in this cluster by ensuring high availability, monitoring and alerting systems.

3. **Alpha Node Cluster**

It is used for pre-production validation and internal testing.
- validating all business logics
- providing QA and integration testing space
- enabling developers to test all comprehensive elements in project, including validating new configurations, CI/CD pipelines, and any infrastructure changes in advance.

4. **Infra Node Cluster**

This cluster is dedicated to infrastructure services that support both the production and alpha clusters.
- Logging system (EFK stack)
- APM (Elastic APM)
- CI/CD tool (Argo CD)
- Secret management (Vault)
- Core supporting services for production application such as databases and Kafka

Especially, by isolating these supporting services from the production cluster, I can reduce the operational workload on the production cluster and improve the overall system stability.