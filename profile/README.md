# MLOps Platform for TRACE Survey Analysis

[![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)]() [![GCP](https://img.shields.io/badge/Google_Cloud-%234285F4.svg?style=for-the-badge&logo=google-cloud&logoColor=white)]() [![Kubernetes](https://img.shields.io/badge/Kubernetes-%23326CE5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)]() [![Docker](https://img.shields.io/badge/Docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)]() [![Jenkins](https://img.shields.io/badge/Jenkins-%23D24939.svg?style=for-the-badge&logo=jenkins&logoColor=white)]() [![Terraform](https://img.shields.io/badge/Terraform-%237B42BC.svg?style=for-the-badge&logo=terraform&logoColor=white)]() [![Helm](https://img.shields.io/badge/Helm-%230F1689.svg?style=for-the-badge&logo=helm&logoColor=white)]() [![Istio](https://img.shields.io/badge/Istio-%23466BB0.svg?style=for-the-badge&logo=istio&logoColor=white)]() [![Go](https://img.shields.io/badge/Go-%2300ADD8.svg?style=for-the-badge&logo=go&logoColor=white)]() [![Python](https://img.shields.io/badge/Python-%233776AB.svg?style=for-the-badge&logo=python&logoColor=white)]() [![PostgreSQL](https://img.shields.io/badge/PostgreSQL-%234169E1.svg?style=for-the-badge&logo=postgresql&logoColor=white)]() [![Kafka](https://img.shields.io/badge/Apache_Kafka-%23231F20.svg?style=for-the-badge&logo=apache-kafka&logoColor=white)]()

## Overview

This repository contains the infrastructure and application code for a comprehensive MLOps platform designed to process and analyze Northeastern University's TRACE course evaluation surveys. The platform spans AWS (for CI/CD infrastructure) and GCP (for Kubernetes workloads and MLOps services), employing GitOps principles with Terraform for infrastructure management and Helm for application deployment.

## Architecture Diagram

![diagram-export-4-19-2025-12_00_12-AM](https://github.com/user-attachments/assets/039c42d8-2b46-4737-afb4-3821132e00de)

## Architecture Explained

The platform is divided into two primary cloud environments: AWS for the CI/CD foundation and GCP for the core MLOps workloads running on GKE.

### Phase 1: AWS - CI/CD Foundation

* **AMI Factory (Packer & GitHub Actions):** A custom Ubuntu AMI is automatically built using Packer whenever changes are merged into the `ami-jenkins` repository. This AMI bundles Jenkins, Nginx/Caddy, and necessary build tools. Builds run in the ROOT AWS account.
* **Jenkins Infrastructure (Terraform):** The `infra-jenkins` Terraform configuration provisions a dedicated VPC and an EC2 instance (from the custom AMI) within the ROOT AWS account (`us-east-1`) to host the Jenkins server. An Elastic IP is assigned for stable access.
* **Jenkins Server:**
    * Acts as the central CI/CD engine.
    * Accessible via `jenkins.jkops.me` (DNS via Route 53), fronted by Nginx/Caddy with Let's Encrypt SSL.
    * Manages automated workflows:
        * **PR Checks:** Validates conventional commit messages and Helm chart syntax/templating using Jenkins jobs.
        * **Build & Push:** Builds versioned Docker images upon code merges and pushes them to a private Docker Hub repository.
        * **Helm Release:** Packages Helm charts, versions them using semantic-release, and creates GitHub Releases.

### Phase 2: GCP - MLOps Infrastructure & Workloads (`us-east-1`)

* **GCP Foundation (Terraform):** The `tf-gcp-infra` Terraform configuration sets up the GCP environment, including:
    * Organization policies, projects (dev/prod/dns).
    * A custom VPC with public (for Bastion) and private (for GKE nodes) subnets.
    * A secure, private, regional GKE cluster (v1.30.9) with COS nodes, Workload Identity, Binary Authorization, and CMEK encryption via Cloud KMS.
    * A Bastion Host VM in the public subnet for secure GKE access.
    * A GCS Bucket for PDF uploads and DB backups.
    * Cloud KMS keys for GKE CMEK and SOPS encryption.
    * A static external IP for ingress.
    * Cloud DNS configuration for application domains.
    * Google Cloud Managed Service for Prometheus for observability.
* **GKE Deployment Process (Helm & SOPS):**
    * Deployments are initiated manually from the Bastion Host.
    * The `helm-charts` repository is cloned.
    * Secrets encrypted within the Helm charts (using SOPS and a Cloud KMS key) are decrypted locally on the Bastion via the `sops` CLI.
    * A script executes `helm install/upgrade` to deploy applications onto GKE using the charts and decrypted values.

### Phase 3: GKE Application Runtime

Applications and services run as containerized workloads within dedicated Kubernetes namespaces on GKE, managed by Helm deployments.

* **Core Services:**
    * **Istio:** Provides a service mesh for secure communication (mTLS), traffic management, and observability. An Istio Ingress Gateway manages external access via the static IP. Sidecars are injected into application pods but excluded from database pods.
    * **Cert-Manager:** Automatically provisions and manages Let's Encrypt TLS certificates for public-facing services exposed via the Istio Ingress Gateway.
    * **Kafka:** Acts as a messaging queue for asynchronous processing between the API server and the Trace Processor.
* **MLOps Applications:**
    * **API Server (`api-server-app` ns):** A Go-based REST API for user interactions, including PDF uploads. Stores PDF metadata in its PostgreSQL DB (`api-server-db` ns) and uploads PDF files to GCS, then publishes processing tasks to Kafka. Exposed at `api.jkops.me`.
    * **Trace Processor (`trace-processor` ns):** Consumes tasks from Kafka, retrieves corresponding PDFs from GCS, processes them, and stores results in its dedicated PostgreSQL DB (`trace-processor-db` ns).
    * **Streamlit Interface (`streamlit-llm-interface` ns):** A Python/Streamlit web application providing a user interface (potentially for querying processed results). Interacts with the API Server via the service mesh. Exposed at `app.jkops.me`.
* **Supporting Services:**
    * **PostgreSQL Instances (`api-server-db`, `trace-processor-db` ns):** StatefulSets providing persistent database storage for the API Server and Trace Processor. Database schemas are managed via Flyway/Liquibase migrations run as init containers. Network Policies restrict access.
    * **DB Backup Operator (`db-backup-operator` ns):** A Kubernetes operator that watches custom resources to perform scheduled backups of specified PostgreSQL schemas to the GCS bucket.
    * **Grafana (`monitoring` ns):** Provides dashboards for visualizing metrics collected by Prometheus. Exposed at `grafana.jkops.me`.
    * **Kiali (`istio-system` ns):** Visualizes the Istio service mesh topology and traffic flow. Exposed at `kiali.jkops.me`.
* **Observability:**
    * **Metrics:** Google Cloud Managed Service for Prometheus scrapes metrics from applications, Istio, Kafka, Postgres exporters, etc., and sends them to Google Cloud Monitoring. Grafana uses Cloud Monitoring as its data source.
    * **Logs:** Application and system logs are shipped to Google Cloud Logging.
    * **Tracing:** Istio enables distributed tracing capabilities.
* **Kubernetes Best Practices:** Deployments utilize HPA & PDBs, unique RBAC Service Accounts, Network Policies, Readiness/Liveness probes, and resource requests/limits.

## Technology Stack

* **Cloud Providers:** AWS, Google Cloud Platform (GCP)
* **Containerization & Orchestration:** Docker, Kubernetes (GKE)
* **CI/CD:** Jenkins, Packer, GitHub Actions (for infra checks), Conventional Commits, Semantic Release
* **Infrastructure as Code:** Terraform
* **Application Deployment:** Helm, SOPS
* **Service Mesh:** Istio
* **Monitoring:** Google Cloud Managed Service for Prometheus, Google Cloud Monitoring, Google Cloud Logging, Grafana, Kiali
* **Messaging:** Apache Kafka
* **Databases:** PostgreSQL
* **Storage:** Google Cloud Storage (GCS)
* **Web Server/Proxy:** Nginx / Caddy
* **Certificate Management:** Cert-Manager, Let's Encrypt
* **Identity & Security:** GCP Workload Identity, GCP Binary Authorization, Cloud KMS, RBAC, Network Policies
* **Languages:** Go (API Server), Python (Streamlit, Trace Processor - *assumption*), Shell Scripting

## Repositories

* `ami-jenkins`: Packer configuration for the Jenkins base AMI.
* `infra-jenkins`: Terraform code for Jenkins AWS infrastructure.
* `tf-gcp-infra`: Terraform code for GCP infrastructure (VPC, GKE, GCS, KMS, etc.).
* `helm-charts`: Helm charts for deploying all applications and services to GKE.
* `api-server`: Go source code for the API server microservice.
* `trace-processor`: Source code for the PDF trace processing microservice.
* `streamlit-llm-interface`: Source code for the Streamlit web UI.
* `db-backup-operator`: Source code for the Kubernetes database backup operator.

## Contributors

Meet the team behind this project:

* **[Karan Thakkar](https://www.linkedin.com/in/thakkaran)**
* **[Jay Gala](https://www.linkedin.com/in/jaygala25)**
