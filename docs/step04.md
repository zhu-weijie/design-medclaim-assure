#### **1. Logical View (C4 Component Diagram)**

This diagram shows the new PDF Processor component listening to events from the scanner and creating the final PNG assets.

```mermaid
C4Component
    title Component Diagram for PDF Processing Service

    System_Boundary(ProcessingBoundary, "Asynchronous Processing") {
        Component(ScanEventBus, "Scan Event Bus", "Pub/Sub Topic", "Broadcasts the results of the virus scan.")
        Component(PDF_Processor, "PDF Processor", "Serverless Function/Container", "Splits PDFs and converts pages to PNGs.")
        Component(ProcessingEventBus, "Processing Event Bus", "Pub/Sub Topic", "Broadcasts that receipt images are ready for verification.")
    }

    System_Boundary(StorageBoundary, "Cloud Storage") {
        Component(CleanStorage, "Clean Storage", "Blob Storage", "Stores files that have passed the virus scan.")
        Component(ProcessedStorage, "Processed Storage", "Blob Storage", "Stores the final, page-by-page PNG images.")
    }

    %% Corrected Relationships Showing Data Flow
    %% The event flows FROM the bus TO the processor
    Rel(ScanEventBus, PDF_Processor, "Triggers with 'Scan Complete' event")

    %% The PDF data flows FROM Clean Storage TO the processor
    Rel(CleanStorage, PDF_Processor, "Provides clean PDF for processing")

    %% The PNG data flows FROM the processor TO Processed Storage
    Rel(PDF_Processor, ProcessedStorage, "Uploads generated PNGs to")

    %% The new event flows FROM the processor TO the next event bus
    Rel(PDF_Processor, ProcessingEventBus, "Publishes 'Processing Complete' event to")

    UpdateLayoutConfig($c4ShapeInRow="2")
```

#### **2. Physical View (AWS Deployment Diagram)**

This diagram shows the physical implementation using AWS Lambda for cost-effective, event-driven compute.

```mermaid
graph TD
    %% Input from the previous stage
    subgraph "Input Event Source"
        SNS_Scan("fa:fa-rss-square SNS Topic<br>(Scan Event Bus)")
    end

    subgraph "PDF Processing Pipeline"
        %% SQS provides a resilient buffer before Lambda
        SQS_PDF("fa:fa-comments SQS Queue<br>(PDF Processing Queue)")
        
        %% Lambda is ideal for event-driven compute
        Lambda("fa:fa-bolt AWS Lambda<br>(PDF Processor)")

        %% Storage buckets
        CleanS3("fa:fa-database S3 Bucket<br>(Clean Storage)")
        ProcessedS3("fa:fa-database S3 Bucket<br>(Processed Storage)")
        
        %% Output for the next stage
        SNS_Process("fa:fa-rss-square SNS Topic<br>(Processing Event Bus)")
    end

    %% Downstream services will subscribe here
    subgraph "Next Steps"
        Placeholder("... (Authenticity Service, etc.)")
    end

    %% Data Flow with corrected labels
    SNS_Scan -- "1 Event (File is clean PDF)" --> SQS_PDF
    SQS_PDF -- "2 Triggers invocation" --> Lambda
    Lambda -- "3 Reads PDF" --> CleanS3
    Lambda -- "4 Writes PNGs" --> ProcessedS3
    Lambda -- "5 Publishes result" --> SNS_Process
    SNS_Process --> Placeholder
```

#### **3. Component-to-Resource Mapping Table**

This table is updated with the new PDF processing components.

| **Logical Component**       | **Physical AWS Resource**                                   | **Rationale for Choice**                                                                                                                                                             |
| :-------------------------- | :---------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Scan Event Bus              | SNS Topic                                                   | No change. This is the event source for our new service.                                                                                                                           |
| Clean Storage               | S3 Bucket                                                   | No change. This is the source of the clean PDF files to be processed.                                                                                                                |
| **PDF Processor** (New)     | **AWS Lambda Function (+ SQS Queue)**                       | **Lambda** is a perfect fit for this task. It's serverless, event-driven, and scales automatically. We pay only for the compute time used. An **SQS Queue** is placed between SNS and Lambda to act as a resilient buffer, handle retries, and enable a dead-letter queue (DLQ) for failed conversions. |
| **Processed Storage** (New) | **S3 Bucket**                                               | S3 is the ideal, scalable, and cost-effective service for storing the generated PNG artifacts.                                                                                    |
| **Processing Event Bus** (New)| **SNS Topic**                                               | SNS provides the pub/sub mechanism to notify any and all downstream services (like the future Authenticity service) that the processed images are ready.                             |
