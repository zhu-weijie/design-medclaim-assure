#### **1. Logical View (C4 Component Diagram)**

The logical components remain the same, but we can now add a Load Balancer to show how traffic is managed.

```mermaid
C4Component
    title Component Diagram for Scalable API and Web Tier

    Person(User, "Authenticated User", "A user of the medical claims system.")
    
    System_Boundary(WebAppBoundary, "Web Application") {
        Component(WebApp, "Web UI", "JavaScript, HTML, CSS", "Provides the file upload interface, delivered via CDN.")
    }
    
    System_Boundary(APIBoundary, "Medical Claims API") {
        Component(LoadBalancer, "API Load Balancer", "Load Balancer", "Distributes traffic across API instances.")
        Component(OrchestrationAPI, "Orchestration API", "Container (Horizontally Scaled)", "Generates pre-signed URLs and publishes events.")
    }

    Rel(User, WebApp, "Accesses application")
    Rel(WebApp, LoadBalancer, "Sends API requests to")
    Rel(LoadBalancer, OrchestrationAPI, "Forwards request to a healthy instance")

    UpdateLayoutConfig($c4ShapeInRow="2", $c4BoundaryInRow="1")
```

#### **2. Physical View (AWS Deployment Diagram)**

This diagram shows the full, production-ready, scalable architecture for the user-facing services.

```mermaid
graph TD
    User("fa:fa-user User")

    subgraph "AWS Cloud"
        %% CDN Layer for the Web UI
        CloudFront("fa:fa-globe CloudFront (CDN)")
        S3_WebApp("fa:fa-database S3 Bucket<br>(Web UI Assets)")
        
        subgraph "VPC (Multi-AZ)"
            subgraph "Public Subnets (AZ-a, AZ-b)"
                APIGateway("fa:fa-server API Gateway")
                ALB("fa:fa-sitemap Application Load Balancer")
            end

            subgraph "Private Subnets (AZ-a, AZ-b)"
                subgraph "ECS/EKS with Auto Scaling"
                    Orchestration_A("fa:fa-box Orchestration Svc<br>(Instance 1)")
                    Orchestration_B("fa:fa-box Orchestration Svc<br>(Instance 2)")
                    Orchestration_N("fa:fa-ellipsis-h ...")
                end
            end
        end
    end
    
    %% Backend services from previous issues remain unchanged
    subgraph "Backend Processing Pipeline"
        SQS("fa:fa-comments SQS Queue")
    end

    %% Data Flow with corrected labels
    User -- "1 Loads website" --> CloudFront
    CloudFront -- "Pulls static content" --> S3_WebApp

    User -- "2 API Calls (e.g., GET /presigned-url)" --> APIGateway
    APIGateway -- "3 Forwards to ALB" --> ALB
    ALB -- "4 Distributes traffic" --> Orchestration_A
    ALB -- " " --> Orchestration_B
    ALB -- " " --> Orchestration_N
    
    Orchestration_A -- "5 Publishes message" --> SQS
    Orchestration_B -- " " --> SQS
```

#### **3. Component-to-Resource Mapping Table**

| **Logical Component**       | **Physical AWS Resource**                                   | **Rationale for Choice**                                                                                                                                                             |
| :-------------------------- | :---------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Web UI** (Updated)        | **S3 Bucket + CloudFront**                                  | Hosting static assets on S3 is extremely cheap and durable. **CloudFront (CDN)** provides low-latency global delivery and caching, significantly improving performance and user experience. |
| **API Load Balancer** (New) | **Application Load Balancer (ALB)**                         | An ALB is a managed Layer 7 load balancer that is ideal for distributing HTTP/HTTPS traffic to containers. It integrates seamlessly with auto-scaling and provides health checks. |
| **Orchestration API** (Updated)| **ECS/EKS Service with Auto Scaling**                     | An ECS/EKS service manages the deployment and health of our containers. **Auto Scaling** allows the service to elastically add or remove container instances based on load, ensuring performance while optimizing cost. |
