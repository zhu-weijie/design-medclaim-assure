#### **1. Logical View (C4 Component Diagram)**

This diagram shows how our generic application components interact with a unified Observability Platform.

```mermaid
C4Component
    title Component Diagram for System Observability

    System_Boundary(MedClaimBoundary, "MedClaim-Assure System") {
        Component(Service, "Application Component", "Container / Function", "Any service within our architecture (e.g., Orchestration API, Scanner, etc.).")
    }
    
    System_Boundary(ObservabilityBoundary, "Observability Platform") {
        Component(Logging, "Centralized Logging", "Log Aggregator", "Collects, stores, and allows searching of all service logs.")
        Component(Metrics, "Metrics & Alarming", "Time-Series Database", "Stores performance metrics and triggers alerts based on thresholds.")
        Component(Tracing, "Distributed Tracing", "Trace Aggregator", "Collects and visualizes end-to-end request traces.")
    }

    Rel(Service, Logging, "Sends structured logs to")
    Rel(Service, Metrics, "Publishes performance metrics to")
    Rel(Service, Tracing, "Sends trace data to")

    UpdateLayoutConfig($c4ShapeInRow="3")
```

#### **2. Physical View (AWS Deployment Diagram)**

This diagram shows how our physical AWS services integrate with the specific AWS observability services.

```mermaid
graph TD
    subgraph "Application Components"
        direction TB
        Orchestration_Services("fa:fa-cubes ECS/EKS Service")
        Lambda_Functions("fa:fa-bolt AWS Lambda Functions")
    end

    subgraph "AWS Observability Suite"
        direction TB
        CloudWatchLogs("fa:fa-align-left CloudWatch Logs")
        CloudWatchMetrics("fa:fa-bar-chart CloudWatch Metrics & Alarms")
        XRay("fa:fa-sitemap AWS X-Ray")
    end
    
    %% Data Flow with corrected labels
    Orchestration_Services -- "1 Pushes Structured Logs" --> CloudWatchLogs
    Lambda_Functions -- " " --> CloudWatchLogs
    
    Orchestration_Services -- "2 Pushes Metrics (CPU, Mem, etc)" --> CloudWatchMetrics
    Lambda_Functions -- " " --> CloudWatchMetrics
    
    Orchestration_Services -- "3 Pushes Trace Data" --> XRay
    Lambda_Functions -- " " --> XRay
```

#### **3. Component-to-Resource Mapping Table**

| **Logical Component**       | **Physical AWS Resource**                                   | **Rationale for Choice**                                                                                                                                                             |
| :-------------------------- | :---------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Centralized Logging**     | **Amazon CloudWatch Logs**                                  | The native, fully managed logging service in AWS. It integrates seamlessly with virtually all other AWS services (ECS, Lambda, API Gateway), making it the default, easiest choice. |
| **Metrics & Alarming**      | **Amazon CloudWatch Metrics & Alarms**                      | The native monitoring service. It automatically collects many essential metrics from AWS resources and provides a robust engine for creating alarms to notify teams via SNS, email, etc. |
| **Distributed Tracing**     | **AWS X-Ray**                                               | The native distributed tracing service in AWS. Its SDKs make it relatively straightforward to instrument our application code, and it provides the end-to-end visibility we need. |
