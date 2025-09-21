#### **1. Logical View (C4 Component Diagram)**

This diagram shows the new asynchronous flow and the introduction of a message queue.

```mermaid
C4Component
  title Component Diagram for Asynchronous Upload Pipeline

  Person(User, "Authenticated User", "A user of the medical claims system.")

  System_Boundary(WebAppBoundary, "Web Application") {
    Component(WebApp, "Web UI", "JavaScript, HTML, CSS", "Manages direct-to-storage uploads and previews.")
  }

  System_Boundary(APIBoundary, "Medical Claims API") {
    Component(OrchestrationAPI, "Orchestration API", "Container (e.g., Node.js/Python)", "Generates pre-signed URLs and publishes events.")
  }

  System_Boundary(StorageBoundary, "Cloud Storage") {
    Component(RawStorage, "Raw Receipt Storage", "Blob Storage", "Temporarily stores unprocessed receipt files uploaded by the user.")
  }

  System_Boundary(ProcessingBoundary, "Asynchronous Processing") {
     Component(ProcessingQueue, "Processing Queue", "Message Queue", "Decouples upload ingestion from backend processing.")
  }

  %% Step-by-step user and system interactions
  Rel(User, WebApp, "Selects files to upload")
  Rel(WebApp, OrchestrationAPI, "1. Requests Pre-Signed URL(s)")
  
  %% This is the critical direct-to-storage relationship
  Rel(WebApp, RawStorage, "2. Uploads file directly using URL")
  
  %% Notification and handoff to the backend
  Rel(WebApp, OrchestrationAPI, "3. Notifies API of upload completion")
  Rel(OrchestrationAPI, ProcessingQueue, "4. Publishes 'File Ready for Processing' message")

  UpdateLayoutConfig($c4ShapeInRow="2", $c4BoundaryInRow="2")
```

#### **2. Physical View (AWS Deployment Diagram)**

This diagram maps the new logical components to their physical AWS counterparts.

```mermaid
graph TD
    subgraph "User's Device"
        User("fa:fa-user User")
    end

    subgraph "AWS Cloud"
        subgraph "VPC"
            subgraph "Public Subnet"
                APIGateway("fa:fa-server API Gateway")
            end

            subgraph "Private Subnet"
                subgraph "Container Orchestration Service (e.g., ECS/EKS)"
                    OrchestrationService("fa:fa-box Orchestration Service Container")
                end
            end
        end

        S3("fa:fa-database S3 Bucket<br>(Raw Receipt Storage)")
        SQS("fa:fa-comments SQS Queue<br>(Processing Queue)")
    end

    %% User Flow for Pre-Signed URL Generation and Upload %%
    User -- "1 GET /presigned-url" --> APIGateway
    APIGateway -- " " --> OrchestrationService
    OrchestrationService -- "Generates URL metadata for S3" --x S3
    OrchestrationService -- "2 Returns URL" --> APIGateway
    APIGateway -- " " --> User

    %% Direct Upload and Asynchronous Notification %%
    User -- "3 PUT file to S3 URL" --> S3
    User -- "4 POST /notify-upload" --> APIGateway
    APIGateway -- " " --> OrchestrationService
    OrchestrationService -- "5 Sends message" --> SQS
```

#### **3. Component-to-Resource Mapping Table**

This table is updated to reflect the new architecture.

| **Logical Component**       | **Physical AWS Resource**                                   | **Rationale for Choice**                                                                                                                                                             |
| :-------------------------- | :---------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Web UI                      | User's Browser                                              | Standard client-side execution environment.                                                                                                                                          |
| Orchestration API           | Container on ECS/EKS                                        | Manages the business logic of generating secure upload URLs and dispatching processing jobs. A container allows for easy scaling and deployment.                                    |
| (Entry Point)               | API Gateway                                                 | Continues to provide a secure, managed entry point for all API interactions.                                                                                                         |
| Raw Receipt Storage         | S3 (Simple Storage Service)                                 | S3's pre-signed URL feature is central to this design. It provides a secure, highly scalable, and direct-to-cloud ingestion point, offloading traffic from our service.              |
| **Processing Queue** (New)  | **SQS (Simple Queue Service)**                              | A fully managed message queue that is highly scalable and durable. It perfectly decouples our API from downstream processing, ensuring resilience and enabling independent scaling. |