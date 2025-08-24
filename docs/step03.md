#### **1. Logical View (C4 Component Diagram)**

This diagram integrates the new Scanner component and shows the file's path based on the scan result.

```mermaid
C4Component
  title Component Diagram for Security Virus Scanning

  System_Boundary(APIBoundary, "Medical Claims API") {
    Component(OrchestrationAPI, "Orchestration API", "Container", "Generates pre-signed URLs and publishes events.")
  }

  System_Boundary(ProcessingBoundary, "Asynchronous Processing") {
    Component(ProcessingQueue, "Processing Queue", "Message Queue", "Holds messages for files awaiting processing.")
    Component(Scanner, "Scanner Service", "Container with ClamAV", "Scans files for viruses and malware.")
    Component(ScanEventBus, "Scan Event Bus", "Pub/Sub Topic", "Broadcasts the results of the virus scan.")
  }

  System_Boundary(StorageBoundary, "Cloud Storage") {
    Component(RawStorage, "1. Raw Storage", "Blob Storage", "Temporarily stores new, un-scanned files.")
    Component(CleanStorage, "2a. Clean Storage", "Blob Storage", "Stores files that have passed the virus scan.")
    Component(QuarantineStorage, "2b. Quarantine Storage", "Blob Storage", "Isolates files that have failed the virus scan.")
  }

  %% Correct Data Flow
  Rel(OrchestrationAPI, ProcessingQueue, "Publishes 'File Ready' message")
  
  %% Scanner PULLS from the queue
  Rel(Scanner, ProcessingQueue, "Consumes 'File Ready' message")

  %% Scanner interacts with storage
  Rel_D(Scanner, RawStorage, "Downloads file from")
  Rel_D(Scanner, CleanStorage, "Moves file to (if clean)")
  Rel_D(Scanner, QuarantineStorage, "Moves file to (if infected)")
  
  %% Scanner PUSHES to the event bus for downstream services
  Rel_D(Scanner, ScanEventBus, "Publishes 'Scan Complete' event to")

  UpdateLayoutConfig($c4ShapeInRow="3")
```

#### **2. Physical View (AWS Deployment Diagram)**

This diagram shows the new physical components integrated into the AWS environment.

```mermaid
graph TD
    %% Handoff from the previous stage
    subgraph " "
     direction LR
     SQS("fa:fa-comments SQS Queue<br>(Processing Queue)")
    end

    subgraph "Security Processing Pipeline (AWS Cloud)"
        direction TB

        %% The core service that does the work
        subgraph "Container Orchestration (ECS/EKS)"
            ScannerService("fa:fa-box Scanner Service<br>(with ClamAV)")
        end

        %% The storage resources it interacts with
        RawS3("fa:fa-database S3 Bucket<br>(Raw Storage)")
        CleanS3("fa:fa-database S3 Bucket<br>(Clean Storage)")
        QuarantineS3("fa:fa-database S3 Bucket<br>(Quarantine Storage)")

        %% The event bus for broadcasting results
        SNS("fa:fa-rss-square SNS Topic<br>(Scan Event Bus)")
    end

    %% Placeholder for the next stages in the architecture
    subgraph "Downstream Services"
     direction LR
     Placeholder("... (PDF Splitter, etc.)")
    end

    %% Define the sequence of operations with corrected labels
    SQS -- "1 Polls for messages" --> ScannerService
    ScannerService -- "2 Downloads file" --> RawS3
    ScannerService -- "3a Moves to Clean Storage" --> CleanS3
    ScannerService -- "3b Moves to Quarantine Storage" --> QuarantineS3
    ScannerService -- "4 Publishes Scan Result" --> SNS
    SNS -- " " --> Placeholder
```

#### **3. Component-to-Resource Mapping Table**

This table is updated to include the new security-focused components.

| **Logical Component**       | **Physical AWS Resource**                                   | **Rationale for Choice**                                                                                                                                                             |
| :-------------------------- | :---------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Orchestration API           | Container on ECS/EKS                                        | No change.                                                                                                                                                                           |
| Raw Receipt Storage         | S3 Bucket                                                   | No change. Lifecycle policies will be added to delete files after a short period (e.g., 24 hours) to ensure this bucket remains temporary.                                         |
| Processing Queue            | SQS                                                         | No change.                                                                                                                                                                           |
| **Scanner Service** (New)   | **Container on ECS/EKS with EFS**                           | A containerized service is ideal for this isolated task. **EFS (Elastic File System)** is added to provide a shared, writable layer for downloading and updating ClamAV virus definitions. |
| **Clean Storage** (New)     | **S3 Bucket**                                               | A new, secure bucket to serve as the source-of-truth for safe files. It will have stricter access controls than the raw bucket.                                                     |
| **Quarantine Storage** (New)| **S3 Bucket**                                               | A highly restrictive bucket. Access will be limited to security administrators only. It will have policies preventing file retrieval and enabling automated deletion after a set period. |
| **Scan Event Bus** (New)    | **SNS (Simple Notification Service)**                         | SNS provides a pub/sub topic, which is perfect for broadcasting the scan result. Multiple downstream services (PDF splitter, notification service, etc.) can subscribe to this topic independently. |
