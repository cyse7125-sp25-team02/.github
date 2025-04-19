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

## System Architecture: PDF Processing & Vectorization

This document outlines the architecture of the system designed to process PDF documents (specifically course feedback reports), extract text, generate vector embeddings, and store them for retrieval-augmented generation (RAG) applications.

### Architecture Diagram

![trace-processor](https://github.com/user-attachments/assets/7ab52078-4329-4d9b-93fb-e2a94abff7e9)

### Workflow Overview

The system follows an event-driven, asynchronous workflow orchestrated using Kafka:

1.  **PDF Upload (API Server):**
    * The process begins when a user or an automated system uploads a PDF file via an API endpoint.

2.  **Message Queuing (Kafka):**
    * Upon successful upload and storage of the PDF (presumably in a persistent store like Google Cloud Storage, although not explicitly shown linked to the API server in this diagram section), the API server publishes a message to a Kafka topic.
    * This message contains metadata about the uploaded file, crucially including its storage path (e.g., GCS URI).

3.  **Worker Consumption (Kafka Consumer):**
    * A dedicated worker service (implemented by the provided Python script) runs as a Kafka consumer, subscribed to the topic.
    * The worker polls the topic and consumes new messages as they arrive.

4.  **PDF Fetching (Worker -> GCS):**
    * Using the path from the Kafka message, the worker fetches the corresponding PDF file from Google Cloud Storage (GCS). *(This step is implied by the `pdf_path` variable in the code and standard practice, linking the Kafka message content to the file needed for processing).*

5.  **OCR Processing (Worker -> Mistral API):**
    * The worker sends the fetched PDF content to the Mistral AI OCR API (`mistralai.Mistral` client's `ocr.process` method in the code).
    * Mistral performs Optical Character Recognition (OCR) and returns the extracted text content structured as Markdown.

6.  **Text Chunking (Worker):**
    * The worker processes the returned Markdown text using the `chunk_feedback_by_qa` function.
    * This function implements a custom logic to parse the Markdown, identify question-answer pairs within the course feedback, clean the text (removing noise like special characters), and split it into meaningful semantic chunks. Each chunk typically represents a single piece of feedback related to a specific question.

7.  **Embedding Generation (Worker -> Vertex AI):**
    * For each text chunk, the worker formats the text along with relevant metadata (course name, instructor, question, etc.) using `create_text_for_embedding`.
    * It then calls the Google Cloud Vertex AI Text Embedding API (`TextEmbeddingModel` in the code, specifically `text-embedding-004`) to generate a high-dimensional vector embedding for each chunk. These embeddings capture the semantic meaning of the text.

8.  **Vector Storage (Worker -> Pinecone):**
    * Finally, the worker takes the generated embeddings, along with the original text and metadata for each chunk, and upserts them into a Pinecone vector database index (`store_chunks_in_pinecone` function).
    * Each record in Pinecone contains the unique chunk ID, the vector embedding, and the associated metadata, making the processed feedback searchable based on semantic similarity.

### Key Components

* **API Server:** Handles incoming PDF uploads and initiates the processing pipeline by publishing to Kafka.
* **Google Cloud Storage (GCS):** Stores the raw PDF files.
* **Kafka:** Acts as a message broker, decoupling the upload process from the intensive processing work and enabling asynchronous operation.
* **Worker (Python Application):** The core processing engine. It consumes messages, interacts with external APIs (Mistral, Vertex AI), performs text manipulation (chunking), and stores results (Pinecone). Likely runs in a containerized environment (e.g., GKE, Cloud Run).
* **Mistral AI API:** Provides OCR capabilities to extract text from PDF documents.
* **Vertex AI Embedding API:** Generates semantic vector embeddings from text chunks.
* **Pinecone:** A managed vector database used to store and index the embeddings for efficient similarity search in downstream RAG applications.

### Technology Stack

* **Programming Language:** Python 3
* **Messaging:** Apache Kafka
* **Cloud Storage:** Google Cloud Storage (GCS)
* **OCR:** Mistral AI API
* **Embeddings:** Google Cloud Vertex AI (Text Embedding Model)
* **Vector Database:** Pinecone
* **Libraries:** `google-cloud-aiplatform`, `vertexai`, `mistralai`, `pinecone-client`, `python-dotenv` (implied for API keys), `tqdm`.

## System Architecture: LLM RAG Streamlit Application

This document describes the architecture of the Streamlit application designed for querying course feedback using a Retrieval-Augmented Generation (RAG) approach powered by Google Gemini, Vertex AI, and Pinecone.

### Architecture Diagram

![LLM-RAG-Streamlit](https://github.com/user-attachments/assets/60d35b7e-cb02-43b8-b0de-ad7b52fa2146)

### Workflow Overview

The application provides a chat interface where users can ask questions about course feedback. The backend leverages a RAG pipeline to generate informed answers:

1.  **User Query Input (Streamlit UI):**
    * The user enters their query (e.g., "How was the instructor for CSYE6225?") into the Streamlit chat interface.

2.  **Intent Check (Streamlit UI):**
    * The application first checks if the input is a simple greeting (e.g., "hello", "thanks"). If so, it provides a canned response and skips the RAG pipeline.

3.  **Query Embedding (Streamlit -> Vertex AI):**
    * If the query is not a simple greeting, the Streamlit backend sends the user's query text to the Google Cloud Vertex AI Text Embedding API (`text-embedding-004` model via the `get_query_embedding` function).
    * Vertex AI computes a vector embedding representing the semantic meaning of the query.

4.  **Vector Search (Streamlit -> Pinecone):**
    * The Streamlit backend takes the generated query embedding and performs a vector similarity search against the pre-populated Pinecone index (`query_pinecone` function).
    * This search retrieves the `TOP_K` feedback chunks from the database whose embeddings are most similar to the query embedding.

5.  **Context Retrieval & Filtering (Pinecone -> Streamlit):**
    * Pinecone returns the most relevant feedback chunks (including their text content and metadata) to the Streamlit application.
    * The application checks if the similarity score of the top match meets a predefined `SIMILARITY_THRESHOLD`. If not, the retrieved context is discarded to avoid using irrelevant information.

6.  **Context Formatting & Prompt Construction (Streamlit):**
    * If relevant context is retrieved (and passes the threshold), the Streamlit backend formats these feedback snippets into a structured context string (`format_context` function).
    * It then constructs a detailed prompt containing the original user query and the formatted context, along with instructions for the LLM on how to generate the answer.

7.  **Answer Generation (Streamlit -> Gemini LLM):**
    * This complete prompt is sent to the Google Gemini LLM (`gemini-1.0-pro` model via the `generate_answer_with_gemini` function).
    * Gemini analyzes the query and the provided context snippets to synthesize a coherent answer based *only* on the retrieved feedback.

8.  **Display Answer (Gemini LLM -> Streamlit -> User):**
    * The generated answer text is returned from Gemini to the Streamlit backend.
    * Streamlit displays the final answer in the chat interface for the user. Optionally, the retrieved context snippets can be viewed in an expander.

### Key Components

* **User:** Interacts with the application via the chat interface.
* **Streamlit UI (Frontend):** Provides the web-based chat interface, handles user input/output, and orchestrates the calls to backend services. The Python script (`app.py`) contains this logic.
* **Vertex AI Embedding Service (Backend):** Generates vector embeddings for user queries.
* **Pinecone (Backend):** A managed vector database storing pre-computed embeddings of course feedback chunks. Used for efficient similarity search.
* **Google Gemini (LLM Backend):** A large language model used to understand the prompt (query + context) and generate a natural language answer.

### Technology Stack

* **Web Framework:** Streamlit
* **Programming Language:** Python 3
* **Embeddings:** Google Cloud Vertex AI (`text-embedding-004`)
* **Vector Database:** Pinecone
* **LLM:** Google Gemini (`gemini-1.0-pro`)
* **Libraries:** `streamlit`, `google-cloud-aiplatform`, `vertexai`, `pinecone-client`, `google-generativeai`, `python-dotenv`.

## Contributors

Meet the team behind this project:

* **[Karan Thakkar](https://www.linkedin.com/in/thakkaran)**
* **[Jay Gala](https://www.linkedin.com/in/jaygala25)**
