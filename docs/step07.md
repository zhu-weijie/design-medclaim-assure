#### **1. Logical View (C4 Component Diagram)**

This diagram shows the `Claims Database` as a central component, now being written to by both the user-facing API and the backend processing services.

```mermaid
C4Component
    title Component Diagram for Data Persistence Layer

    System_Boundary(APIBoundary, "Medical Claims API") {
        Component(OrchestrationAPI, "Orchestration API", "Container", "Creates initial Claim/File records and generates pre-signed URLs.")
    }

    System_Boundary(ProcessingBoundary, "Asynchronous Processing") {
        Component(AuthenticitySvc, "Authenticity Service", "Function/Container", "Updates final verification status for Claims/Files.")
    }

    System_Boundary(StorageBoundary, "Cloud Storage & Database") {
        ComponentDb(ClaimsDB, "Claims Database", "Database", "Single source of truth for claim and file metadata and status.")
    }

    %% Relationships
    Rel(OrchestrationAPI, ClaimsDB, "Writes initial claim data to")
    Rel(AuthenticitySvc, ClaimsDB, "Updates final claim status in")

    UpdateLayoutConfig($c4ShapeInRow="2")
```

#### **2. Physical View (AWS Deployment Diagram)**

This diagram shows both the `Orchestration Service` and the `Authenticity Service` now interacting with the same DynamoDB table.

```mermaid
graph TD
    subgraph "User-Facing Tier"
        Orchestration_Services("fa:fa-cubes ECS/EKS Service<br>(Orchestration API)")
    end

    subgraph "Backend Processing Tier"
        Lambda_Auth("fa:fa-bolt AWS Lambda<br>(Authenticity Service)")
    end
    
    subgraph "Shared Data Layer"
        DynamoDB("fa:fa-table DynamoDB Table<br>(Claims Database)")
    end

    %% Data Flow with corrected labels
    Orchestration_Services -- "1 Creates Claim & File records<br>(Status: PENDING_UPLOAD)" --> DynamoDB
    
    %% This comment represents the time gap where the async pipeline runs
    
    Lambda_Auth -- "2 Updates Claim & File records<br>(Status: VERIFIED / REJECTED)" --> DynamoDB
```

#### **3. Component-to-Resource Mapping Table**

No new physical resources are introduced, but the responsibilities of existing components are updated.

| **Logical Component**       | **Physical AWS Resource**                                   | **Rationale for Choice (with Updates)**                                                                                                                                                             |
| :-------------------------- | :---------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Orchestration API           | ECS/EKS Service with Auto Scaling                           | **Responsibility Update:** In addition to generating pre-signed URLs, this service is now responsible for the initial creation of claim and file records in DynamoDB. This makes the upload process stateful. |
| Authenticity Service        | AWS Lambda Function                                         | **Responsibility Update:** Instead of just performing analysis, this service now has the critical final step of updating the claim and file records in DynamoDB with the final verification status and extracted metadata. |
| **Claims Database**         | **Amazon DynamoDB**                                         | It is now the central state management service for the entire application lifecycle. Its serverless nature, high availability, and low-latency key-value access make it ideal for this role. |
