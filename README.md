# Lovable – AI Coding SaaS

Cloud-native AI coding platform that transforms natural-language requirements into contextualized software artifacts through a modular, event-driven Spring Boot microservice architecture.

<div align="center">
    <img src="https://img.shields.io/badge/Java-21-orange" alt="Java 21">
    <img src="https://img.shields.io/badge/Spring_Boot-3.x-green" alt="Spring Boot">
    <img src="https://img.shields.io/badge/Spring_AI-Integrated-blue" alt="Spring AI">
    <img src="https://img.shields.io/badge/Kafka-Event_Driven-black" alt="Kafka">
    <img src="https://img.shields.io/badge/Kubernetes-GKE-blue" alt="Kubernetes">
</div>

## Engineering Highlights

- **Multi-Service Spring Boot Architecture:** Separation of concerns across Account, Workspace, and Intelligence services.
- **Agentic AI Context Gathering:** Tool-based dynamic retrieval of file trees and content directly into LLM prompts without needing static RAG.
- **Asynchronous File Writes:** AI-generated edits are dispatched as Kafka events (`FileStoreRequestEvent`) for non-blocking persistence.
- **Real-Time Response Streaming:** Reactive Spring WebFlux (`Flux`) and Server-Sent Events (SSE) for low-latency AI response streaming.
- **Centralized Infrastructure:** Netflix Eureka for service discovery, Spring Cloud Config for configuration, and Spring Cloud Gateway for routing.
- **Kubernetes Deployment:** Automated GKE deployment via GitHub Actions (Jib-based Docker builds).
- **15+ REST APIs:** Handling project workspaces, authentication, file storage, and AI interactions.

### Reported Project Benchmark
> Indexed 10,000+ documents with approximately 120 ms p95 retrieval latency under the project's benchmark environment. *(Note: This benchmark represents the external load capacity of the retrieval system and is not directly reproducible from the codebase without the benchmark dataset).*

## Executive Overview

Building an AI coding platform is more than simply sending text to an LLM API. The backend must coordinate user state, project workspaces, dynamic context gathering, asynchronous file edits, and service-to-service communication.

This project solves these engineering challenges through a cloud-native microservice architecture. When a user requests a code change, the Intelligence Service uses Agentic function calling (Spring AI Tools) to dynamically read necessary project files from the Workspace Service. The AI's response is streamed back to the client in real-time, while file edits are asynchronously parsed and published to a Kafka topic. The Workspace service independently consumes these events and persists the changes, ensuring fast API response times and strict fault isolation.

## Key Features

- **Agentic AI Code Generation:** Spring AI integrated with `CodeGenerationTools` to selectively read workspace files based on LLM decision-making.
- **Real-Time Streaming:** Text-event streams (`TEXT_EVENT_STREAM_VALUE`) via Spring WebFlux.
- **Event-Driven File Persistence:** Kafka-backed asynchronous file writes to decouple LLM generation from I/O operations.
- **Project & Workspace Management:** Endpoints for creating projects, adding members, and managing file trees.
- **API Gateway & Service Discovery:** Spring Cloud Gateway and Netflix Eureka for dynamic routing and internal load balancing.
- **Centralized Configuration:** Spring Cloud Config pointing to a remote Git repository for dynamic configuration updates.
- **Automated CI/CD:** GitHub Actions workflows for compiling, containerizing with Jib, and deploying to Google Kubernetes Engine (GKE).

## Architecture

The system utilizes a modern, distributed microservices pattern:

```mermaid
flowchart TD
    Client[Client] -->|HTTP / SSE| Gateway[API Gateway]

    Gateway -->|Routes Requests| Account[Account Service]
    Gateway -->|Routes Requests| Workspace[Workspace Service]
    Gateway -->|Routes Requests| Intelligence[Intelligence Service]

    subgraph "Core Microservices"
        Account
        Workspace
        Intelligence
    end
    
    Intelligence <-->|Fetches File Tree / Content| Workspace
    Intelligence -->|Publishes File Edits| Kafka((Kafka Broker))
    Kafka -->|Consumes File Edits| Workspace
    
    Workspace -->|Stores Files / Previews| Minio[(MinIO / Storage)]
    Account --> DB_Account[(PostgreSQL: account_db)]
    Workspace --> DB_Workspace[(PostgreSQL: workspace_db)]
    Intelligence --> DB_Intel[(PostgreSQL: intelligence_db)]

    subgraph "Infrastructure Services"
        Config[Config Service]
        Discovery[Discovery Service - Eureka]
    end
    
    Core Microservices -.->|Fetch Config| Config
    Core Microservices -.->|Register/Discover| Discovery
```

### Service Responsibility Table

| Component | Responsibility |
|---|---|
| **API Gateway** | Single entry point, JWT authentication filtering, and request routing to downstream services. |
| **Account Service** | User authentication, JWT issuance, and Stripe billing integration. |
| **Workspace Service** | Project management, role-based access control, file tree management, and Kafka event consumption for file storage. |
| **Intelligence Service** | Spring AI integration, LLM prompt augmentation, agentic tool execution, and response streaming. |
| **Discovery Service** | Netflix Eureka server for internal service registration and discovery. |
| **Config Service** | Spring Cloud Config server reading centralized configuration from a remote Git repository. |
| **Common Library** | Shared DTOs, Kafka event schemas, JWT utilities, and security configurations. |

## AI & Context Pipeline

The AI workflow avoids traditional vector-RAG in favor of an **Agentic Context Pipeline**.

1. **User Request:** The client sends a chat request to the Intelligence Service.
2. **Context Augmentation:** `FileTreeContextAdvisor` intercepts the prompt and synchronously fetches the project's file tree from the Workspace Service, injecting it as a System Message.
3. **Agentic File Reading:** The LLM evaluates the file tree and utilizes `CodeGenerationTools` (Spring AI Function Calling) to request the contents of specific relevant files.
4. **Streaming Inference:** The LLM generates the response, which is streamed back to the client via WebFlux Server-Sent Events.
5. **Asynchronous Parsing:** Upon completion, the full response is parsed for file edits (`ChatEvent`).
6. **Event Dispatch:** File edits are packaged into `FileStoreRequestEvent`s and published to Kafka.
7. **Workspace Update:** The Workspace Service consumes the events and persists the generated code.

## Technology Choices

| Technology | Role | Why it fits this system |
|---|---|---|
| **Java 21 & Spring Boot 3** | Backend Framework | Provides a robust, type-safe foundation with virtual threads and deep Spring ecosystem integration. |
| **Spring AI** | AI Orchestration | Simplifies LLM integration, function calling (Tools), and streaming (Flux) without relying on external Python services. |
| **Kafka** | Event Streaming | Decouples the Intelligence service from the Workspace service, preventing I/O bottlenecks during generation. |
| **Netflix Eureka** | Service Discovery | Enables dynamic scaling and location transparency for internal service-to-service communication. |
| **Spring Cloud Gateway** | API Routing | Provides a reactive, non-blocking gateway to handle SSE streaming and centralized JWT validation. |
| **Kubernetes & Docker** | Orchestration | Ensures each microservice is independently deployable, scalable, and isolated. |

## Kubernetes Deployment

The platform is designed for cloud-native deployment on GKE. The `k8s/` directory contains complete manifests:

- **Stateful Sets:** PostgreSQL (with `pgvector` image prepared), Redis, Kafka, and MinIO.
- **Deployments & Services:** Individual deployments for every Spring Boot microservice.
- **Network Policies:** Granular ingress/egress isolation across namespaces (`lovable-core`).
- **Secrets Management:** Centralized `app-secrets` for database passwords and API keys.
- **Ingress:** Nginx Ingress routing for external API exposure.

*Deployment is fully automated via GitHub Actions utilizing Jib for rootless, daemonless Docker builds.*

## CI/CD Pipeline

The repository utilizes GitHub Actions (`.github/workflows/`) for automated CI/CD:
- **Trigger:** Push to `master` containing changes to a specific service directory.
- **Build:** Compiles the `common-lib` and the target microservice using Maven.
- **Containerization:** Utilizes Google's Jib Maven Plugin to build optimized Docker images without a Docker daemon.
- **Deployment:** Authenticates with Google Cloud, retrieves GKE credentials, updates the Kubernetes Deployment with the new image SHA, and waits for rollout completion.

## Project Structure

```text
.
├── account-service/        # Auth, Users, and Billing
├── api-gateway/            # Spring Cloud Gateway & JWT filters
├── common-lib/             # Shared DTOs, Security, and Event Models
├── config-service/         # Spring Cloud Config Server
├── discovery-service/      # Netflix Eureka Server
├── intelligence-service/   # Spring AI, Tool Calling, WebFlux SSE
├── k8s/                    # Kubernetes Manifests (Infra, Services, Stateful)
│   ├── infra/              # Namespaces, Ingress, Network Policies
│   ├── services/           # Microservice Deployments
│   └── stateful/           # Kafka, MinIO, PostgreSQL, Redis
├── workspace-service/      # Project state, Files, Kafka Consumers
└── .github/workflows/      # CI/CD Deployment Pipelines
```

## Local Development

### Prerequisites
- Java 21
- Maven
- Docker (for stateful services like Kafka/PostgreSQL)
- OpenAI API Key

### Setup Instructions

1. **Clone and Build Common Library:**
   ```bash
   git clone https://github.com/nikhilanand0105/Lovable-AI-Coding-SaaS.git
   cd Lovable-AI-Coding-SaaS
   cd common-lib && ./mvnw clean install
   ```

2. **Start Infrastructure (Requires Docker):**
   *(Note: Utilize the provided Kubernetes manifests or deploy local instances of Kafka, PostgreSQL, and Eureka).*

3. **Configure Environment Variables:**
   Set the following variables in your environment or IDE:
   ```env
   OPENAI_API_KEY=your-api-key
   CONFIG_SERVER_URL=http://localhost:8888
   ```

4. **Service Startup Order:**
   1. `discovery-service` (Eureka)
   2. `config-service` (Config Server)
   3. `api-gateway`
   4. `account-service`, `workspace-service`, `intelligence-service` (Any order)

## API Documentation Reference

*Partial list of verifiable endpoints extracted from controllers:*

| Service | Method | Endpoint | Purpose |
|---|---|---|---|
| Account | `POST` | `/auth/signup` | Register a new user |
| Workspace | `POST` | `/projects` | Create a new project workspace |
| Workspace | `GET` | `/projects/{id}` | Retrieve project details |
| Workspace | `GET` | `/projects/{projectId}/files/tree` | Get workspace file structure |
| Intelligence | `POST` | `/chat/stream` | Stream AI response with Agentic tool calling |

## Engineering Decisions & Trade-offs

### Agentic File Reading vs. Vector Search
Instead of chunking the codebase and placing it into a Vector Database, this project utilizes **Agentic Tool Calling**. The LLM receives the directory tree and explicitly requests the files it needs.
*Trade-off:* This ensures 100% accurate file context and avoids RAG chunking artifacts, but it increases the prompt token size and LLM reasoning time compared to pure similarity search. (The `pgvector` image is provisioned in Kubernetes, indicating a future hybrid approach).

### Event-Driven File Persistence
File generation is decoupled using Kafka (`FileStoreRequestEvent`).
*Trade-off:* This prevents the `intelligence-service` from blocking on I/O operations while streaming, providing a fast interactive UX. However, it introduces eventual consistency; the user might see the streamed response slightly before the file is fully persisted in the workspace.

## Future Roadmap & Hardening Opportunities

While the architecture is highly capable, the following areas represent planned future improvements:
- **Vector-Based RAG Implementation:** Utilize the existing `pgvector` Kubernetes StatefulSet to implement hybrid retrieval (Agentic + Vector Search) for massive codebases.
- **Testing Coverage:** Expand beyond context-load tests to include comprehensive unit, integration, and contract testing for service boundaries.
- **Observability:** Introduce OpenTelemetry, distributed tracing (Zipkin/Jaeger), and Prometheus/Grafana metrics.
- **Kubernetes Autoscaling:** Implement HorizontalPodAutoscaler (HPA) manifests to dynamically scale services under load.
- **Enhanced Security:** Implement service-to-service authentication (e.g., mTLS) and stricter rate limiting at the API Gateway.
