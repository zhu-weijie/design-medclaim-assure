#### **1. Logical View (C4 Component Diagram)**

This diagram shows security as a cross-cutting concern applied to our components.

```mermaid
C4Component
    title Component Diagram for End-to-End Security Concepts

    System_Boundary(MedClaimBoundary, "MedClaim-Assure System") {
        Component(SecureComponent, "Secure Application Component", "Container / Function", "Any service, now operating with strict, role-based permissions.")
    }
    
    System_Boundary(AWS_Boundary, "AWS Environment") {
        Component(SecureResource, "Secure AWS Resource", "S3, DynamoDB, SQS", "Any AWS resource, with data encrypted at rest and in transit.")
    }

    Rel(SecureComponent, SecureResource, "Accesses via Secure Network (Security Groups & VPC Endpoints) with Least-Privilege IAM Role")

    UpdateLayoutConfig($c4ShapeInRow="2")
```

#### **2. Physical View (AWS Deployment Diagram)**

This is the final, complete production architecture, augmented to show all the security layers working together.

```mermaid
graph TD
    User("fa:fa-user User")

    subgraph "AWS Cloud"
        CloudFront("fa:fa-globe CloudFront (CDN)")
        
        subgraph "VPC (Multi-AZ)"
            subgraph " "
                direction LR
                S3_WebApp("fa:fa-database S3<br>(Web UI Assets)<br>Encrypted")
            end

            subgraph "Public Subnets"
                sg_alb("fa:fa-shield Security Group")
                ALB("fa:fa-sitemap Application Load Balancer")
                sg_alb -- "Allows Port 443 In" --> ALB
            end

            subgraph "Private Subnets"
                sg_ecs("fa:fa-shield Security Group")
                subgraph "ECS/EKS with Auto Scaling"
                    Orchestration_Services("fa:fa-cubes Orchestration Svc<br>(IAM Role)")
                end
                sg_ecs -- "Allows traffic from ALB SG" --> Orchestration_Services
                
                %% Backend Lambda services
                sg_lambda("fa:fa-shield Security Group")
                Lambda_Functions("fa:fa-bolt Lambda Functions<br>(IAM Role)")
                sg_lambda -- "Internal traffic only" --> Lambda_Functions
                
                %% VPC Endpoints
                VPCE_S3("fa:fa-lock VPC Endpoint<br>for S3")
                VPCE_SQS("fa:fa-lock VPC Endpoint<br>for SQS")
                VPCE_Dynamo("fa:fa-lock VPC Endpoint<br>for DynamoDB")
                
                Orchestration_Services -- "via VPCe" --> VPCE_SQS
                Lambda_Functions -- "via VPCe" --> VPCE_S3
                Lambda_Functions -- "via VPCe" --> VPCE_Dynamo
            end
        end
    end
    
    %% User Traffic Flow
    User -- "HTTPS (TLS)" --> CloudFront
    CloudFront --> S3_WebApp
    User -- "HTTPS (TLS)" --> ALB
    ALB -- "TLS" --> Orchestration_Services
```

#### **3. Component-to-Resource Mapping Table (Security Focus)**

| **Security Principle**      | **Physical AWS Resource / Feature**                         | **Rationale for Choice**                                                                                                                                                             |
| :-------------------------- | :---------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Encryption at Rest**      | **AWS KMS-Managed Keys** on S3 Buckets & DynamoDB Table     | Using AWS-managed keys (SSE-KMS) provides a simple, secure, and auditable way to enforce encryption on all data stores without the complexity of managing our own encryption keys.   |
| **Encryption in Transit**   | **TLS Certificates on CloudFront & ALB**                    | AWS Certificate Manager (ACM) provides free, auto-renewing TLS certificates that are easily integrated with CloudFront and ALBs, ensuring all user-facing traffic is encrypted. |
| **Least Privilege**         | **IAM Roles for ECS Tasks & Lambda Functions**              | IAM Roles provide temporary, fine-grained credentials to our compute resources. Defining these with minimal permissions (e.g., only access to a specific S3 prefix) is the core of least privilege. |
| **Network Firewall**        | **Security Groups**                                         | Security Groups are stateful firewalls that allow us to create explicit allow-rules for traffic between components, effectively creating a zero-trust network environment.        |
| **Private Networking**      | **VPC Endpoints** (Gateway for S3, Interface for others)    | VPC Endpoints ensure that traffic from our services to other AWS APIs stays within the AWS network, enhancing security by avoiding the public internet.                       |
