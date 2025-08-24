### **Architecture-as-Code (AaC)**

#### **1. Logical View (C4 Component Diagram)**

This diagram shows the primary logical components and their interactions for the initial MVP.

```mermaid
C4Component
  title Component Diagram for Core File Upload MVP

  Person(User, "Authenticated User", "A user of the medical claims system.")
  System_Boundary(WebAppBoundary, "Web Application") {
    Component(WebApp, "Web UI", "JavaScript, HTML, CSS", "Provides the file upload interface.")
  }
  System_Boundary(APIBoundary, "Medical Claims API") {
    Component(UploadAPI, "Upload API", "Container (e.g., Node.js/Python)", "Receives files and handles the upload logic.")
  }
  System_Boundary(StorageBoundary, "Cloud Storage") {
    Component(ReceiptStorage, "Receipt Storage", "Blob Storage", "Stores the raw, unprocessed receipt files.")
  }

  Rel(User, WebApp, "Uploads a receipt file")
  Rel(WebApp, UploadAPI, "Sends file via HTTPS POST request")
  Rel(UploadAPI, ReceiptStorage, "Writes file to")

  UpdateLayoutConfig($c4ShapeInRow="2", $c4BoundaryInRow="2")
```

#### **2. Physical View (AWS Deployment Diagram)**

This diagram maps the logical components to a potential physical implementation on AWS using a container-based approach.

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
                    UploadServiceContainer("fa:fa-box Upload Service Container")
                end
            end

            S3("fa:fa-database S3 Bucket<br>(Receipt Storage)")
        end
    end

    User -- "HTTPS" --> APIGateway
    APIGateway -- "Forwards Request" --> UploadServiceContainer
    UploadServiceContainer -- "Uploads File" --> S3
```

#### **3. Component-to-Resource Mapping Table**

This table explicitly connects the logical components to the chosen physical AWS resources.

| **Logical Component** | **Physical AWS Resource**                                   | **Rationale for Choice**                                                                                                                              |
| :-------------------- | :---------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Web UI                | User's Browser                                              | Standard client-side execution environment. The UI will be served from a service like S3/CloudFront, but the code runs on the user's machine.      |
| Upload API            | Container on ECS/EKS                                        | A containerized approach is specified. ECS or EKS provide managed orchestration, allowing for future scalability, though we start with a single instance. |
| (Entry Point)         | API Gateway                                                 | Provides a secure, managed entry point for our API. It handles routing, rate limiting, and can offload concerns like TLS termination.           |
| Receipt Storage       | S3 (Simple Storage Service)                                 | Highly durable, scalable, and cost-effective object storage service, perfect for storing user-uploaded files like images and PDFs.                   |
