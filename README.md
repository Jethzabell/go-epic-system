# go-epic-system

# Distributed Media Processing Pipeline: Complete System Design

This document outlines the complete system design for a **Distributed Media Processing Pipeline**. This architecture leverages modern microservices principles and integrates the following components:

- **gRPC API Service** for job submission and status queries
- **REST Gateway with Swagger UI (via Echo)** for interactive API documentation and a user-friendly front end
- **Kafka** as the high-throughput message broker
- **Worker Services** for processing media using **ffmpeg**
- **PostgreSQL** for persistent storage
- **Redis** for caching job statuses
- **Kubernetes** for container orchestration and scaling

---

## 1. Overview

The system accepts media processing jobs from clients, processes them asynchronously, and provides real-time status updates. It is designed to be scalable, resilient, and easy to maintain.

---

## 2. Architecture Diagram

```plaintext
                       +---------------------+
                       |      Clients        |
                       |  (gRPC & REST API)  |
                       +----------+----------+
                                  |
                 +----------------+----------------+
                 |                                 |
                 v                                 v
      +---------------------+          +------------------------+
      | Swagger UI / REST   |          |    gRPC API Service    |
      | (Echo Framework)    |          |  - Validates Requests  |
      | - Interactive docs  |          |  - Writes to PostgreSQL|
      | - API Testing Tool  |          |  - Updates Redis Cache |
      +---------+-----------+          |  - Publishes to Kafka  |
                 |                      +-----------+------------+
                 +-----------+----------------------+  
                             |                      
                             v                      
                         +----------+     
                         | Kafka    |  (Message Bus)
                         +----------+     
                             |
                             v
                    +---------------------+
                    |  Worker Service(s)  |
                    |  - Consumes Kafka   |
                    |    messages         |
                    |  - Processes media  |
                    |    with ffmpeg      |
                    |  - Updates PostgreSQL|
                    |    & Redis          |
                    +----------+----------+
                               |
             +-----------------+-----------------+
             |                                   |
             v                                   v
      +---------------+                  +---------------+
      | PostgreSQL    |                  |   Redis       |
      | (Persistent   |                  | (Caching Job  |
      |  Storage)     |                  |  Statuses)    |
      +---------------+                  +---------------+

## 3. Component Details & Interactions

### A. Clients & Front End

- **Clients:**  
  - Interact with the system via gRPC or a RESTful API.
  - Use Swagger UI for exploring and testing API endpoints interactively.

- **Swagger UI / REST Gateway (Echo Framework):**  
  - **Role:**  
    - Provides interactive API documentation.
    - Serves as a REST gateway (via tools like grpc-gateway) to translate REST calls into gRPC requests.
  - **Implementation Example (Echo):**
    ```go
    import (
        "github.com/labstack/echo/v4"
        echoSwagger "github.com/swaggo/echo-swagger"
    )

    func main() {
        e := echo.New()

        // Define REST endpoints which may internally call gRPC APIs.
        // Example: e.GET("/jobs/:id", getJobStatus)

        // Mount Swagger UI at /swagger/*
        e.GET("/swagger/*", echoSwagger.WrapHandler)

        e.Start(":8080")
    }
    ```

### B. gRPC API Service

- **Responsibilities:**
  - **Job Submission:**  
    - Receives media processing job requests via gRPC.
    - Validates and enriches the job request.
  - **Persistence & Caching:**  
    - Writes job metadata (job ID, submission time, parameters) to **PostgreSQL**.
    - Stores an initial status (e.g., "queued") in **Redis**.
  - **Message Publishing:**  
    - Publishes job messages to **Kafka** for asynchronous processing.
- **Concurrency:**  
  - Leverages Go’s goroutines and channels to handle multiple simultaneous requests efficiently.

### C. Kafka (Message Broker)

- **Responsibilities:**
  - **Decoupling:**  
    - Acts as the intermediary between the API Service and Worker Services.
  - **Reliability & Scalability:**  
    - Provides durable message storage and supports consumer groups.
- **Workflow:**
  - The gRPC API Service produces messages to a designated Kafka topic.
  - Worker Services consume messages from this topic for processing.

### D. Worker Service

- **Responsibilities:**
  - **Message Consumption:**  
    - Subscribes to the Kafka topic to retrieve job messages.
  - **Media Processing:**  
    - Processes each job by invoking **ffmpeg** (using Go’s `os/exec` package).
  - **Data Updates:**  
    - Updates job details in **PostgreSQL** with processing results.
    - Refreshes the job status in **Redis** (e.g., "processing", "completed", "failed").
- **Concurrency:**  
  - Implements a worker pool pattern using goroutines and channels.
  - **Example Worker Function:**
    ```go
    func worker(id int, jobs <-chan Job, results chan<- Result) {
        for job := range jobs {
            result := processJobWithFFmpeg(job)
            updateJobStatusInRedis(job.ID, result.Status)
            saveJobResultToPostgres(job.ID, result)
            results <- result
        }
    }
    ```

### E. Data Storage & Caching

- **PostgreSQL:**
  - Persists all structured data including job metadata, processing logs, and user information.
  - Provides ACID guarantees for critical data.
- **Redis:**
  - Acts as an in-memory cache for job statuses.
  - Implements TTL (Time To Live) for transient data to prevent stale entries.

### F. Kubernetes (Container Orchestration)

- **Responsibilities:**
  - **Container Deployment:**  
    - All microservices (gRPC API, REST/Swagger UI, Worker Services) are containerized.
  - **Scaling & Management:**  
    - Uses Deployments, ReplicaSets, and Services for lifecycle management and load balancing.
    - Implements Horizontal Pod Autoscaler (HPA) to scale based on load (e.g., Kafka lag, CPU/memory usage).
  - **Configuration & Secrets:**  
    - Manages configuration data via ConfigMaps and sensitive information via Secrets (e.g., Kafka, PostgreSQL, Redis credentials).

---

## 4. Data Flow Walkthrough

### A. Job Submission & API Gateway Flow

1. **Client Request:**  
   - A client submits a media processing job via the Swagger UI or a direct gRPC call.

2. **REST Gateway (Swagger UI / Echo):**  
   - If using REST, the gateway converts the REST call into a gRPC request.

3. **gRPC API Service Processing:**  
   - Validates the job.
   - Writes job metadata to **PostgreSQL**.
   - Sets an initial job status in **Redis**.
   - Publishes a job message to **Kafka**.

4. **Kafka:**  
   - Receives and persists the job message for processing by Worker Services.

### B. Job Processing Flow

1. **Worker Service Consumption:**  
   - Worker instances (part of a Kafka consumer group) pick up job messages.

2. **Media Processing:**  
   - Each worker invokes **ffmpeg** to process the media file (e.g., transcoding, thumbnail extraction).

3. **Data Update:**  
   - After processing, workers update the job record in **PostgreSQL** and refresh the status in **Redis**.

### C. Job Status Query Flow

1. **Client Query:**  
   - A client queries the job status via the Swagger UI or directly through a gRPC/REST endpoint.

2. **API Service Response:**  
   - The service retrieves the current job status from **Redis** for fast response.
   - Detailed information can be fetched from **PostgreSQL** if required.

---

## 5. Scaling, Fault Tolerance & Monitoring

### Scaling

- **Horizontal Scaling:**  
  - Scale the gRPC API and Worker Services independently based on load.
  - Kafka consumer groups allow multiple worker instances to share the processing load.
  
- **Caching Efficiency:**  
  - Redis minimizes the load on PostgreSQL by handling frequent status queries.

### Fault Tolerance

- **Message Durability:**  
  - Kafka provides persistent logs and consumer offset management to ensure no message loss.
  
- **Container Resilience:**  
  - Kubernetes automatically restarts failed pods and manages rolling updates.
  
- **Retry Logic:**  
  - Worker Services implement retries and error handling for transient failures.

### Monitoring & Logging

- **Metrics:**  
  - Use Prometheus and Grafana to monitor key system metrics (e.g., Kafka lag, CPU, memory).
  
- **Centralized Logging:**  
  - Integrate with logging solutions (such as the ELK stack) to collect and analyze logs from all services.
  
- **Alerting:**  
  - Configure alerts for job failures, high latency, or service disruptions.

---

## 6. Summary

This system design provides a robust, scalable, and decoupled architecture for media processing:

- **gRPC API Service** handles efficient job management.
- **REST/Swagger UI (Echo)** offers interactive API documentation and testing.
- **Kafka** decouples job submission from processing, ensuring high throughput.
- **Worker Services** leverage Go’s concurrency patterns and **ffmpeg** for media processing.
- **PostgreSQL** stores structured, persistent data.
- **Redis** offers fast, in-memory caching for job statuses.
- **Kubernetes** orchestrates containerized services for high availability and scalability.

This design demonstrates best practices in API design, message-driven architectures, real-time processing, and cloud-native deployments.
