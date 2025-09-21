### **Project Requirement Document: MedClaim-Assure**

*   **Version:** v1.0.0
*   **Date:** 21 September 2025

### 1. Executive Summary

MedClaim-Assure is a cloud-native system designed to automate and secure the medical claims submission process. The project's core objective is to replace manual, error-prone workflows with a highly efficient, scalable, and secure web application. This will be achieved by providing an intuitive file upload interface for users and a powerful, automated backend pipeline that performs security scanning, document processing, and authenticity verification. This document specifies the functional and non-functional requirements for the successful delivery of Version v1.0.0 of the system.

### 2. Goals & Objectives

*   **To automate** receipt processing, reducing manual verification time by over 90%.
*   **To enhance** system and user security by eliminating the risk of malicious file uploads.
*   **To strengthen** fraud detection by implementing automated, AI-driven receipt analysis.
*   **To deliver** a superior user experience through a responsive, reliable, and high-performance submission process.
*   **To establish** a highly available and scalable cloud architecture capable of supporting future growth.

### 3. Functional Requirements (FRs)

#### **FR-1: User-Facing File Upload**
*   **FR-1.1: File Ingestion**
    *   **FR-1.1.1: File Constraints:** The system shall accept file uploads of the following types: `image/jpeg`, `image/png`, `application/pdf`. The maximum size per file shall be **10MB**.
    *   **FR-1.1.2: Concurrent Uploads:** The system shall support the simultaneous upload of up to **10 files** for a single claim submission.
*   **FR-1.2: User Interface Feedback**
    *   **FR-1.2.1: Upload State:** The UI shall display a distinct progress indicator for each file being uploaded.
    *   **FR-1.2.2: Image Previews:** Upon successful upload, the UI shall display a thumbnail preview for each successfully uploaded image or for each page of a successfully processed PDF.
*   **FR-1.3: Client-Side Error Handling**
    *   **FR-1.3.1:** If a user attempts to upload a file that violates the constraints in FR-1.1.1, the UI shall present a clear error message without initiating an upload.
    *   **FR-1.3.2:** If a file upload fails in transit, the UI shall clearly mark the specific file as failed and allow the user to retry it individually.

#### **FR-2: Backend Processing Pipeline**
*   **FR-2.1: Security Scanning**
    *   **FR-2.1.1: Quarantine Protocol:** Upon detection of a virus or malware, the system shall immediately move the infected file to a secure quarantine storage location. The file's status in the database shall be updated to `REJECTED_INFECTED`, and it shall be excluded from any further processing.
*   **FR-2.2: PDF-to-Image Conversion**
    *   **FR-2.2.1: Artifact Generation:** For each multi-page PDF that passes the security scan, the system shall convert every page into a separate, sequentially numbered PNG file. The original PDF must be retained and linked to its derivative PNGs in the database.
*   **FR-2.3: Authenticity Verification**
    *   **FR-2.3.1: Verification Statuses:** The authenticity service shall analyze each receipt image and assign it one of the following explicit statuses:
        *   `VERIFIED`: The image is deemed authentic.
        *   `REJECTED_FRAUDULENT`: The image shows clear signs of tampering.
        *   `NEEDS_MANUAL_REVIEW`: The analysis was inconclusive.
    *   **FR-2.3.2: Metadata Extraction:** For all images that are not rejected, the system shall extract and store key-value data, including but not limited to: `vendor_name`, `transaction_date`, and `total_amount`.

#### **FR-3: Data Persistence and State Management**
*   **FR-3.1: State Lifecycle:** The system shall track the status of each file and the parent claim through the following explicit lifecycle states. No other states are permitted in v1.0.0.
    | Status                  | Trigger                                       |
    | :---------------------- | :-------------------------------------------- |
    | `PENDING_UPLOAD`        | Claim initiated by user; pre-signed URL requested. |
    | `UPLOAD_COMPLETE`       | User's browser confirms direct upload to S3 is complete. |
    | `SCANNING`              | File processing message consumed by Scanner Service. |
    | `PROCESSING`            | File is clean and is being processed (e.g., PDF split). |
    | `PENDING_VERIFICATION`  | Processed images are awaiting authenticity analysis. |
    | `VERIFIED`              | Authenticity analysis passed.               |
    | `NEEDS_MANUAL_REVIEW`   | Authenticity analysis was inconclusive.       |
    | `REJECTED_INFECTED`     | Virus scan failed.                            |
    | `REJECTED_FRAUDULENT`   | Authenticity analysis failed.                 |
*   **FR-3.2: Data Model:** The system shall maintain three core data entities with the following relationships: a `User` can have many `Claims`, and a `Claim` can have many `Files`.

### 4. Non-Functional Requirements (NFRs)

#### **NFR-1: Performance & Latency**
*   **NFR-1.1: System Load:** The system shall meet all performance targets while under a simulated load of **300 concurrent users**, 5,000 daily active users, and 12,000 daily file uploads.
*   **NFR-1.2: API Response Time:** For all synchronous, user-facing API endpoints (e.g., pre-signed URL generation), the server-side response time shall be:
    *   p95 (95th percentile) < 3 seconds.
    *   p97 (97th percentile) < 5 seconds.
*   **NFR-1.3: Backend Processing SLO:** The end-to-end processing time for a claim (from `UPLOAD_COMPLETE` to a final status) shall be less than **5 minutes** for 99% of all submissions.

#### **NFR-2: Availability**
*   **NFR-2.1: Uptime SLA:** The user-facing API and web application shall have a measured uptime of **99.9%** or higher.
*   **NFR-2.2: Disaster Recovery:** The system must be able to withstand the failure of a single AWS Availability Zone without data loss and with minimal interruption to service.

#### **NFR-3: Security**
*   **NFR-3.1: Data-at-Rest Encryption:** All data stored in S3 and DynamoDB shall be encrypted using AWS KMS-managed keys (SSE-KMS).
*   **NFR-3.2: Data-in-Transit Encryption:** All network traffic, both external and internal within the VPC, shall be encrypted using TLS 1.2 or higher.
*   **NFR-3.3: Least Privilege (IAM):** Each compute component shall be assigned a unique IAM Role. The associated IAM policy must not use wildcards (`*`) and must only grant the specific permissions required for that service's explicit function (e.g., the PDF Processor role will have read-only access to the `Clean` bucket and write-only access to the `Processed` bucket).
*   **NFR-3.4: Network Isolation:** Components shall be deployed in private subnets unless they absolutely require public internet access. All inter-service communication shall be governed by Security Groups with explicit allow-rules, defaulting to a deny-all posture.

#### **NFR-4: Observability**
*   **NFR-4.1: Logging:** All logs shall be in a structured JSON format. Logs must be retained for a minimum of **90 days**.
*   **NFR-4.2: Monitoring & Alarming:** Alarms shall be configured for all key SLOs/SLIs (e.g., API latency, error rate, queue depth). Alarm notifications shall be sent to a designated PagerDuty service for the on-call team.
*   **NFR-4.3: Traceability:** Every request originating from the user-facing API shall be assigned a unique Correlation ID, which must be propagated to and included in the logs of all downstream services that participate in processing that request.

### 5. Scope

#### **In-Scope:**
*   All requirements explicitly listed in sections 3 and 4.
*   The complete, automated infrastructure-as-code (IaC) required to deploy and manage the system.

#### **Out-of-Scope:**
*   User authentication/authorization systems (the system will integrate with a pre-existing identity provider).
*   User-facing dashboards for viewing claim history or status.
*   Administrative dashboards for manual review of claims.
*   A user notification system (e.g., email/SMS alerts).
*   A data analytics and reporting pipeline for business intelligence.
*   A native mobile application.
