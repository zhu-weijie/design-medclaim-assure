#### **1. Logical View (C4 Component Diagram)**

This diagram introduces the Authenticity Service, its external AI dependency, and the final Claims Database.

```mermaid
C4Component
    title Component Diagram for Authenticity Verification Service

    System_Boundary(ProcessingBoundary, "Asynchronous Processing") {
        Component(ProcessingEventBus, "Processing Event Bus", "Pub/Sub Topic", "Broadcasts that receipt images are ready.")
        Component(AuthenticitySvc, "Authenticity Service", "Serverless Function/Container", "Orchestrates receipt analysis with an external AI service.")
    }

    System_Boundary(StorageBoundary, "Cloud Storage") {
        Component(ProcessedStorage, "Processed Storage", "Blob Storage", "Stores the final PNG receipt images.")
        ComponentDb(ClaimsDB, "Claims Database", "Database", "Stores claim status and extracted receipt metadata.")
    }
    
    System_Ext(AI_Service, "External AI Service", "3rd Party System", "Provides OCR and document analysis capabilities.")

    %% Corrected Relationships Showing Data Flow
    Rel(ProcessingEventBus, AuthenticitySvc, "Triggers with 'Processing Complete' event")
    
    %% Data flows FROM storage TO the service
    Rel(ProcessedStorage, AuthenticitySvc, "Provides PNG images for analysis")

    %% Data flows FROM the service TO the external system and back
    Rel(AuthenticitySvc, AI_Service, "Sends images for analysis and receives results via API")
    
    %% Data flows FROM the service TO the database
    Rel(AuthenticitySvc, ClaimsDB, "Writes verification status and metadata to")

    UpdateLayoutConfig($c4ShapeInRow="2")
```

#### **2. Physical View (AWS Deployment Diagram)**

This diagram shows the physical implementation, including Amazon Textract as the AI service and DynamoDB as the database.

```mermaid
graph TD
    %% Input from the previous stage
    subgraph "Input Event Source"
        SNS_Process("fa:fa-rss-square SNS Topic<br>(Processing Event Bus)")
    end

    subgraph "Authenticity Verification Pipeline"
        %% SQS provides a resilient buffer
        SQS_Auth("fa:fa-comments SQS Queue<br>(Authenticity Queue)")
        
        %% Lambda for the service logic
        Lambda_Auth("fa:fa-bolt AWS Lambda<br>(Authenticity Service)")

        %% Final data store
        DynamoDB("fa:fa-table DynamoDB Table<br>(Claims Database)")
        
        %% Source image store
        ProcessedS3("fa:fa-database S3 Bucket<br>(Processed Storage)")
    end
    
    %% External AI Service
    subgraph "External AWS AI Service"
        Textract("fa:fa-magic Amazon Textract")
    end

    %% Data Flow with corrected S3 read direction
    SNS_Process -- "1 Event (Images are ready)" --> SQS_Auth
    SQS_Auth -- "2 Triggers invocation" --> Lambda_Auth
    
    %% CORRECTED FLOW: Data flows FROM S3 TO Lambda
    ProcessedS3 -- "3 Reads PNGs" --> Lambda_Auth
    
    Lambda_Auth -- "4 Sends images for analysis" --> Textract
    Textract -- "5 Returns analysis results" --> Lambda_Auth
    Lambda_Auth -- "6 Writes claim status" --> DynamoDB
```

#### **3. Component-to-Resource Mapping Table**

This final table for our processing pipeline includes the authenticity components.

| **Logical Component**       | **Physical AWS Resource**                                   | **Rationale for Choice**                                                                                                                                                             |
| :-------------------------- | :---------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Processing Event Bus        | SNS Topic                                                   | No change. This is the event source for our new service.                                                                                                                           |
| Processed Storage           | S3 Bucket                                                   | No change. This is the source of the final PNG images to be analyzed.                                                                                                                |
| **Authenticity Service** (New) | **AWS Lambda Function (+ SQS Queue)**                       | A serverless function is perfect for this event-driven, stateless task that orchestrates calls to other services. The SQS queue provides resilience and a DLQ for failed analyses. |
| **External AI Service** (New) | **Amazon Textract**                                         | A managed AWS service that specializes in OCR and extracting data from documents like receipts. It's a natural fit, keeping the solution within the AWS ecosystem.             |
| **Claims Database** (New)     | **Amazon DynamoDB**                                         | A fully managed, serverless NoSQL database. It offers excellent performance for the expected workload (writing/reading claim status by a key) and scales seamlessly.                 |
