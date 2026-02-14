# Design Document: Container Verification and Certification System

## 1. Executive Summary

This document presents the architectural design for a cloud-native Container Verification and Certification System deployed on AWS. The system addresses critical challenges faced by global shipping ports during peak operational hours, where thousands of container verification requests must be processed efficiently without delays, downtime, or resource bottlenecks.

### Problem Statement

Global shipping ports process thousands of container verification and certification requests during peak operational hours. Existing on-premise or statically provisioned systems are unable to efficiently handle sudden traffic surges, leading to delayed container clearance, system downtime, and poor resource utilization. Moreover, centralized databases frequently become performance bottlenecks, further degrading system responsiveness and operational efficiency.

### Solution Overview

The proposed solution leverages AWS cloud infrastructure to deliver:
- **Elastic Scalability**: Automatic scaling to handle traffic surges from 100 to 10,000+ requests per minute
- **High Availability**: Multi-region deployment with 99.99% uptime SLA
- **Distributed Architecture**: Elimination of database bottlenecks through Aurora PostgreSQL with read replicas
- **Sub-second Response Times**: Optimized processing pipeline ensuring rapid container clearance
- **Zero Downtime**: Active-standby failover and rolling deployments

## 2. Architecture Overview

### 2.1 High-Level Architecture


The system follows a multi-region, cloud-native microservices architecture deployed on AWS:

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│                                    AWS Cloud                                       │
│                                                                                 ___|   
│  ┌─────────────────────────────────────┐  ┌────────────────────────────────    │
│  │      Region A (Primary)             │  │      Region B (DR)              │  │
│  │      us-east-1                      │  │      eu-west-1                  │  │
│  │                                     │  │                                 │  │
│  │  ┌────────────────────────────────┐ │  │ ┌────────────────────────────┐  │  │
│  │  │  Amazon Route 53               │ │  │ │  Amazon Route 53           │  │  │
│  │  │  (Global DNS + Health Checks)  │◄┼──┼─┤  (Failover Routing)        │  │  │
│  │  └──────────────┬─────────────────┘ │  │ └──────────┬─────────────────┘  │  │
│  │                 │                   │  │            │                    │  │
│  │  ┌──────────────▼─────────────────┐ │  │ ┌──────────▼─────────────────┐  │  │
│  │  │         VPC (10.0.0.0/16)      │ │  │ │    VPC (10.1.0.0/16)       │  │  │
│  │  │                                │ │  │ │                            │  │  │
│  │  │  ┌─────────────────────────┐   │ │  │ │  ┌─────────────────────┐   │  │  │
│  │  │  │  Public Subnet (AZ-1)   │   │ │  │ │  │  Public Subnet      │   │  │  │
│  │  │  │  10.0.1.0/24            │   │ │  │ │  │  10.1.1.0/24        │   │  │  │
│  │  │  │                         │   │ │  │ │  │                     │   │  │  │
│  │  │  │  ┌──────────────────┐   │   │ │  │ │  │  ┌──────────────┐   │   │  │  │
│  │  │  │  │ Network Load     │   │   │ │  │ │  │  │ Network Load │   │   │  │  │
│  │  │  │  │ Balancer (NLB)   │   │   │ │  │ │  │  │ Balancer     │   │   │  │  │
│  │  │  │  │ Layer 4          │   │   │ │  │ │  │  └──────┬───────┘   │   │  │  │
│  │  │  │  └────────┬─────────┘   │   │ │  │ │  └─────────┼───────────┘   │  │  │
│  │  │  └───────────┼─────────────┘   │ │  │ └────────────┼───────────────┘  │  │
│  │  │              │                 │ │  │              │                  │  │
│  │  │  ┌───────────▼─────────────┐   │ │  │  ┌───────────▼─────────────┐    │  │
│  │  │  │  Private Subnet (AZ-1)  │   │ │  │  │  Private Subnet         │    │  │
│  │  │  │  10.0.10.0/24           │   │ │  │  │  10.1.10.0/24           │    │  │
│  │  │  │                         │   │ │  │  │                         │    │  │
│  │  │  │  ┌──────────────────┐   │   │ │  │  │  ┌──────────────────┐   │    │  │
│  │  │  │  │  Amazon EKS      │   │   │ │  │  │  │  Amazon EKS      │   │    │  │
│  │  │  │  │  Cluster         │   │   │ │  │  │  │  Cluster         │   │    │  │
│  │  │  │  │                  │   │   │ │  │  │  │                  │   │    │  │
│  │  │  │  │  ┌────────────┐  │   │   │ │  │  │  │  ┌────────────┐  │   │    │  │
│  │  │  │  │  │ API Gateway│  │   │   │ │  │  │  │  │ API Gateway│  │   │    │  │
│  │  │  │  │  │ Pods (3-10)│  │   │   │ │  │  │  │  │ Pods       │  │   │    │  │
│  │  │  │  │  └────────────┘  │   │   │ │  │  │  │  └────────────┘  │   │    │  │
│  │  │  │  │  ┌────────────┐  │   │   │ │  │  │  │  ┌────────────┐  │   │    │  │
│  │  │  │  │  │Verification│  │   │   │ │  │  │  │  │Verification│  │   │    │  │
│  │  │  │  │  │Service Pods│  │   │   │ │  │  │  │  │Service Pods│  │   │    │  │
│  │  │  │  │  │(3-50 HPA)  │  │   │   │ │  │  │  │  │(Auto-scale)│  │   │    │  │
│  │  │  │  │  └────────────┘  │   │   │ │  │  │  │  └────────────┘  │   │    │  │
│  │  │  │  │  ┌────────────┐  │   │   │ │  │  │  │  ┌────────────┐  │   │    │  │
│  │  │  │  │  │Certification│ │   │   │ │  │  │  │  │Certification│ │   │    │  │
│  │  │  │  │  │Service Pods│  │   │   │ │  │  │  │  │Service Pods│  │   │    │  │
│  │  │  │  │  │(2-30 HPA)  │  │   │   │ │  │  │  │  │(Auto-scale)│  │   │    │  │
│  │  │  │  │  └────────────┘  │   │   │ │  │  │  │  └────────────┘  │   │    │  │
│  │  │  │  └──────────────────┘   │   │ │  │  │  └──────────────────┘   │    │  │
│  │  │  └─────────────────────────┘   │ │  │  └─────────────────────────┘    │  │
│  │  │                                │ │  │                                 │  │
│  │  │  ┌─────────────────────────┐   │ │  │  ┌─────────────────────────┐    │  │
│  │  │  │  Amazon MQ (ActiveMQ)   │   │ │  │  │  Amazon MQ (Standby)    │    │  │
│  │  │  │  Active Broker          │◄──┼─┼──┼──┤  Failover Broker        │    │  │
│  │  │  │  AZ-1 + AZ-2            │   │ │  │  │                         │    │  │
│  │  │  └─────────────────────────┘   │ │  │  └─────────────────────────┘    │  │
│  │  │                                │ │  │                                 │  │
│  │  │  ┌─────────────────────────┐   │ │  │  ┌─────────────────────────┐    │  │
│  │  │  │  Amazon Aurora          │   │ │  │  │  Amazon Aurora          │    │  │
│  │  │  │  PostgreSQL             │   │ │  │  │  Read Replica           │    │  │
│  │  │  │                         │   │ │  │  │  (Cross-Region)         │    │  │
│  │  │  │  ┌──────────────────┐   │   │ │  │  │                         │    │  │
│  │  │  │  │ Writer (AZ-1)    │   │   │ │  │  │  ┌──────────────────┐   │    │  │
│  │  │  │  └──────────────────┘   │   │ │  │  │  │ Reader Instance  │   │    │  │
│  │  │  │  ┌──────────────────┐   │◄──┼─┼──┼──┤  └──────────────────┘   │    │  │
│  │  │  │  │ Reader 1 (AZ-1)  │   │   │ │  │  │                         │    │  │
│  │  │  │  └──────────────────┘   │   │ │  │  │  Async Replication      │    │  │
│  │  │  │  ┌──────────────────┐   │   │ │  │  │  (< 1 sec lag)          │    │  │
│  │  │  │  │ Reader 2 (AZ-2)  │   │   │ │  │  │                         │    │  │
│  │  │  │  └──────────────────┘   │   │ │  │  └─────────────────────────┘    │  │
│  │  │  └─────────────────────────┘   │ │  │                                 │  │
│  │  │                                │ │  │                                 │  │
│  │  │  ┌─────────────────────────┐   │ │  │  ┌─────────────────────────┐    │  │
│  │  │  │  Amazon S3              │   │ │  │  │  Amazon S3              │    │  │
│  │  │  │  Document Storage       │◄──┼─┼──┼──┤  (Cross-Region          │    │  │
│  │  │  │  Versioning Enabled     │   │ │  │  │   Replication)          │    │  │
│  │  │  └─────────────────────────┘   │ │  │  └─────────────────────────┘    │  │
│  │  │                                │ │  │                                 │  │
│  │  │  ┌─────────────────────────┐   │ │  │                                 │  │
│  │  │  │  Amazon CloudWatch      │   │ │  │                                 │  │
│  │  │  │  Logs + Metrics         │   │ │  │                                 │  │
│  │  │  │  X-Ray Tracing          │   │ │  │                                 │  │
│  │  │  └─────────────────────────┘   │ │  │                                 │  │
│  │  └────────────────────────────────┘ │  └────────────────────────────────-┘  │
│  └─────────────────────────────────────┘                                       │
│                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                    Starburst Stargate (Data Federation)                  │  │
│  │                    Cross-Region Query Federation                         │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────-┘
```

### 2.2 AWS Service Mapping

| Component | AWS Service | Purpose | Scaling Strategy |
|-----------|-------------|---------|------------------|
| DNS & Routing | Amazon Route 53 | Global traffic management, health checks, failover | N/A (Managed) |
| Load Balancer | Network Load Balancer (NLB) | Layer 4 load balancing, high throughput | Auto-scaling targets |
| API Gateway | Amazon EKS Pods | Request validation, authentication, routing | HPA: 3-10 pods |
| Verification Service | Amazon EKS Pods | Container verification logic | HPA: 3-50 pods |
| Certification Service | Amazon EKS Pods | Certification generation | HPA: 2-30 pods |
| Message Queue | Amazon MQ (ActiveMQ) | Request queuing, decoupling | Active-Standby failover |
| Database | Amazon Aurora PostgreSQL | Certification storage, audit logs | 1 writer + 2-15 readers |
| Document Storage | Amazon S3 | Container documentation | Auto-scaling (managed) |
| Container Orchestration | Amazon EKS | Kubernetes management | Cluster Autoscaler |
| Monitoring | CloudWatch + X-Ray | Metrics, logs, distributed tracing | N/A (Managed) |
| Data Federation | Starburst Enterprise | Cross-region query federation | On-demand scaling |
| Networking | Amazon VPC | Network isolation, security | N/A |



### 2.3 Architectural Principles

1. **Elastic Scalability**: All compute resources (EKS pods, Aurora replicas) scale automatically based on demand
2. **Multi-Region Resilience**: Active-passive deployment across two AWS regions for disaster recovery
3. **Managed Services First**: Leverage AWS managed services to reduce operational overhead
4. **Network Isolation**: VPC with public/private subnet separation for security
5. **Queue-Based Decoupling**: Amazon MQ decouples request ingestion from processing
6. **Distributed Data**: Aurora PostgreSQL with multi-AZ replication eliminates database bottlenecks
7. **Observability**: Comprehensive monitoring with CloudWatch and X-Ray for full system visibility
8. **Zero Downtime**: Rolling deployments, health checks, and automatic failover ensure continuous operation

## 3. Detailed Component Design

### 3.1 Network Architecture

#### VPC Configuration

**Primary Region (us-east-1)**:
```
VPC CIDR: 10.0.0.0/16

Public Subnets (Internet-facing):
  - AZ-1: 10.0.1.0/24 (NLB, NAT Gateway)
  - AZ-2: 10.0.2.0/24 (NLB, NAT Gateway)

Private Subnets (Application tier):
  - AZ-1: 10.0.10.0/24 (EKS worker nodes)
  - AZ-2: 10.0.11.0/24 (EKS worker nodes)

Private Subnets (Data tier):
  - AZ-1: 10.0.20.0/24 (Aurora, Amazon MQ)
  - AZ-2: 10.0.21.0/24 (Aurora, Amazon MQ)
```

**DR Region (eu-west-1)**:
```
VPC CIDR: 10.1.0.0/16

Public Subnets:
  - AZ-1: 10.1.1.0/24

Private Subnets:
  - AZ-1: 10.1.10.0/24 (EKS worker nodes)
  - AZ-2: 10.1.11.0/24 (EKS worker nodes)
```

#### Network Components

**Internet Gateway**: Provides internet access for public subnets

**NAT Gateways**: 
- One per availability zone in primary region
- Enables private subnet resources to access internet for updates
- High availability with automatic failover

**VPC Endpoints**:
- S3 Gateway Endpoint (no data transfer charges)
- Secrets Manager Interface Endpoint
- CloudWatch Logs Interface Endpoint
- ECR Interface Endpoint (for container image pulls)

**VPC Peering**: 
- Cross-region VPC peering between primary and DR regions
- Enables private communication for replication traffic

#### Security Groups

**NLB Security Group**:
```
Inbound Rules:
  - Port 443 (HTTPS) from 0.0.0.0/0
  - Port 80 (HTTP) from 0.0.0.0/0 (redirect to 443)

Outbound Rules:
  - Port 8080 to EKS Security Group
```

**EKS Worker Node Security Group**:
```
Inbound Rules:
  - Port 8080 from NLB Security Group
  - Port 443 from EKS Control Plane
  - All traffic from same security group (pod-to-pod)

Outbound Rules:
  - Port 5671 (AMQPS) to Amazon MQ Security Group
  - Port 5432 (PostgreSQL) to Aurora Security Group
  - Port 443 to VPC Endpoints
  - Port 443 to 0.0.0.0/0 (for external APIs)
```

**Aurora Security Group**:
```
Inbound Rules:
  - Port 5432 from EKS Security Group

Outbound Rules:
  - None (database doesn't initiate outbound connections)
```

**Amazon MQ Security Group**:
```
Inbound Rules:
  - Port 5671 (AMQPS) from EKS Security Group
  - Port 8162 (Web Console) from VPN/Bastion only

Outbound Rules:
  - None
```

### 3.2 Amazon EKS Cluster Configuration

#### Cluster Specifications

**Control Plane**:
- Managed by AWS (no management overhead)
- Multi-AZ deployment for high availability
- Kubernetes version: 1.28 (latest stable)
- Private endpoint access enabled
- Public endpoint access disabled (access via VPN/bastion)

**Worker Nodes**:
```
Node Group 1 (On-Demand - Baseline):
  Instance Type: c5.2xlarge (8 vCPU, 16 GB RAM)
  Min Nodes: 3
  Max Nodes: 10
  Desired: 3
  AMI: Amazon EKS Optimized AMI
  Disk: 100 GB gp3 (encrypted)

Node Group 2 (Spot - Burst Capacity):
  Instance Type: c5.2xlarge, c5.4xlarge
  Min Nodes: 0
  Max Nodes: 20
  Desired: 0
  Spot Strategy: Capacity-optimized
  Interruption Handling: Enabled
```

#### Cluster Autoscaler Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler
data:
  scale-down-enabled: "true"
  scale-down-les persistent volume support
- Encrypted EBS volumes for stateful workloads

**CoreDNS**:
- Internal DNS resolution for services
- Scaled based on cluster size

**kube-proxy**:
- Network proxy for service communication
- iptables mode for performance

**VPC CNI**:
- Native VPC networking for pods
- Each pod gets VPC IP address
- Security group per pod support

### 3.3 Microservices Architecture

#### 3.3.1 API Gateway Service

**Responsibility**: Entry point for all external requests

**Deployment Configuration**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
  template:
    spec:
      containers:
      - name: api-gateway
        image: container-registry/api-gateway:v1.0
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

**Horizontal Pod Autoscaler**:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**API Endpoints**:
```
POST /api/v1/verification/submit
  Description: Submit container verification request
  Request Body: VerificationRequest (JSON)
  Response: 202 Accepted with requestId
  Rate Limit: 100 requests/minute per user

GET /api/v1/verification/{requestId}/status
  Description: Check verification status
  Response: VerificationStatus (JSON)
  
GET /api/v1/certification/{certificationId}
  Description: Retrieve certification document
  Response: Certification (JSON)
  
POST /api/v1/certification/{certificationId}/revoke
  Description: Revoke a certification
  Request Body: RevocationReason (JSON)
  Response: 200 OK
  
GET /api/v1/audit/logs
  Description: Query audit logs
  Query Parameters: startDate, endDate, containerId, userId
  Response: AuditLogCollection (JSON)
  Authorization: Admin role required
```

**Key Operations**:
1. **Authentication**: Validate JWT tokens issued by AWS Cognito
2. **Authorization**: Check user roles and permissions
3. **Rate Limiting**: Enforce per-user rate limits using Redis
4. **Request Validation**: Validate JSON schema and required fields
5. **Routing**: Forward requests to appropriate backend services
6. **Error Handling**: Return standardized error responses



#### 3.3.2 Verification Service

**Responsibility**: Validate container documentation and contents against international shipping regulations

**Deployment Configuration**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: verification-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: verification-service
  template:
    spec:
      serviceAccountName: verification-service-sa
      containers:
      - name: verification-service
        image: container-registry/verification-service:v1.0
        resources:
          requests:
            cpu: 2000m
            memory: 4Gi
          limits:
            cpu: 4000m
            memory: 8Gi
        env:
        - name: AMAZON_MQ_ENDPOINT
          valueFrom:
            secretKeyRef:
              name: mq-credentials
              key: endpoint
        - name: AURORA_ENDPOINT
          valueFrom:
            secretKeyRef:
              name: aurora-credentials
              key: reader-endpoint
        - name: S3_BUCKET
          value: container-docs-primary-us-east-1
```

**Horizontal Pod Autoscaler**:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: verification-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: verification-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: External
    external:
      metric:
        name: amazonmq_queue_depth
      target:
        type: AverageValue
        averageValue: "100"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
```

**Verification Logic**:

1. **Document Completeness Check**:
   - Required documents: Bill of Lading, Customs Declaration, Safety Certificate, Packing List
   - Each document must be present and readable
   - Return specific error for each missing document

2. **Document Authenticity Validation**:
   - Verify digital signatures using public key infrastructure
   - Check document issuer against authorized issuer list
   - Validate documentl inspection
   - Apply machine learning model for anomaly detection

**Processing Flow**:
```
1. Dequeue request from Amazon MQ
2. Fetch documents from S3 using pre-signed URLs
3. Run parallel validation checks
4. Aggregate results
5. If PASSED: Send to Certification Service
6. If FAILED: Log failure reason and notify submitter
7. Update request status in Aurora
8. Acknowledge message in Amazon MQ
```

**Error Handling**:
- Transient errors (network timeout): Retry up to 3 times with exponential backoff
- Validation errors: Move to dead-letter queue and notify administrator
- Document not found: Return specific error to user

#### 3.3.3 Certification Service

**Responsibility**: Generate and manage certification documents

**Deployment Configuration**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: certification-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: certification-service
  template:
    spec:
      serviceAccountName: certification-service-sa
      containers:
      - name: certification-service
        image: container-registry/certification-service:v1.0
        resources:
          requests:
            cpu: 1000m
            memory: 2Gi
          limits:
            cpu: 2000m
            memory: 4Gi
```

**Horizontal Pod Autoscaler**:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: certification-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: certification-service
  minReplicas: 2
  maxReplicas: 30
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**Key Operations**:

1. **Generate Certification**:
   - Create unique certification ID (UUID v4)
   - Generate digital signature using AWS KMS
   - Set expiration date (90 days from issuance)
   - Store certification in Aurora
   - Generate PDF document and store in S3
   - Send notification to submitter

2. **Retrieve Certification**:
   - Query Aurora by certification ID
   - Return certification details
   - Generate pre-signed S3 URL for PDF download

3. **Revoke Certification**:
   - Update certification status to REVOKED
   - Log revocation reason
   - Send notification to relevant parties
   - Update audit trail

**Certification Format**:
```json
{
  "certificationId": "550e8400-e29b-41d4-a716-446655440000",
  "containerId": "ABCU1234567",
  "requestId": "req-123456",
  "issuedAt": "2026-02-14T10:30:00Z",
  "expiresAt": "2026-05-15T10:30:00Z",
  "issuedBy": "Port Authority - New York",
  "issuingOfficer": "John Doe",
  "verificationResult": {
    "status": "PASSED",
    "checks": [
      {
        "checkType": "DOCUMENT_COMPLETENESS",
        "passed": true,
        "details": "All required documents present"
      },
      {
        "checkType": "MANIFEST_VERIFICATION",
        "passed": true,
        "details": "Contents match manifest"
      }
    ]
  },
  "digitalSignature": "SHA256:abc123...",
  "status": "ACTIVE"
}
```

### 3.4 Amazon MQ (Message Queue)

**Configuration**:
```
Broker Engine: ActiveMQ 5.17
Deployment Mode: Active/Standby
Instance Type: mq.m5.large (2 vCPU, 8 GB RAM)
Storage: 200 GB EBS (gp3, encrypted)
Multi-AZ: Yes (Primary in AZ-1, Standby in AZ-2)
```

**Queue Configuration**:
```xml
<policyEntry queue="verification.requests">
  <deadLetterStrategy>
    <individualDeadLetterStrategy queuePrefix="DLQ." />
  </deadLetterStrategy>
  <pendingQueuePolicy>
    <storeCursor />
  </pendingQueuePolicy>
</policyEntry>
```

**Key Features**:

1. **Persistent Messages**: All messages persisted to EBS before acknowledgment
2. **FIFO Ordering**: Messages processed in order of submission
3. **Dead Letter Queue**: Failed messages moved to DLQ after 3 retry attempts
4. **Automatic Failover**: Standby broker promoted in < 1 minute
5. **Message TTL**: Messages expire after 24 hours if not processed

**Retry Logic**:
```
Attempt 1: Immediate processing
Attempt 2: Retry after 1 second
Attempt 3: Retry after 2 seconds
Attempt 4: Retry after 4 seconds
After 3 failures: Move to DLQ
```

**Monitoring Metrics**:
- Queue depth (target: < 1000 messages)
- Message age (target: < 30 seconds)
- Consumer lag (target: < 10 seconds)
- Dead letter queue depth (alert if > 10)

### 3.5 Amazon Aurora PostgreSQL

**Configuration**:
```
Engine: Aurora PostgreSQL 15.4
Instance Class: 
  - Writer: db.r6g.2xlarge (8 vCPU, 64 GB RAM)
  - Readers: db.r6g.xlarge (4 vCPU, 32 GB RAM)
Storage: Auto-scaling (10 GB to 128 TB)
Multi-AZ: Yes
Backup Retention: 35 days
Encryption: AES-256 (AWS KMS)
```

**Cluster Topology**:
```
Primary Region (us-east-1):
  - 1 Writer instance (AZ-1)
  - 2 Reader instances (AZ-1, AZ-2)
  - Auto-scaling: Up to 15 readers based on CPU

DR Region (eu-west-1):
  - 1 Cross-region read replica
  - Asynchronous replication (< 1 second lag)
  - Can be promoted to writer for failover
```

**Database Schema**:

```sql
-- Certifications table
CREATE TABLE certifications (
    certification_id UUID PRIMARY KEY,
    container_id VARCHAR(11) NOT NULL,
    request_id UUID NOT NULL,
    issued_at TIMESTAMP NOT NULL,
    expires_at TIMESTAMP NOT NULL,
    issued_by VARCHAR(255) NOT NULL,
    issuing_officer VARCHAR(255) NOT NULL,
    verification_result JSONB NOT NULL,
    digital_signature TEXT NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_certifications_container_id ON certifications(container_id);
CREATE INDEX idx_certifications_status ON certifications(status);
CREATE INDEX idx_certifications_issued_at ON certifications(issued_at);

-- Audit logs table
CREATE TABLE audit_logs (
    log_id UUID PRIMARY KEY,
    timestamp TIMESTAMP NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    actor_id VARCHAR(255) NOT NULL,
    container_id VARCHAR(11),
    request_id UUID,
    certification_id UUID,
    outcome VARCHAR(50) NOT NULL,
    details JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_logs_timestamp ON audit_logs(timestamp);
CREATE INDEX idx_audit_logs_event_type ON audit_logs(event_type);
CREATE INDEX idx_audit_logs_container_id ON audit_logs(container_id);

-- Verification requests table
CREATE TABLE verification_requests (
    request_id UUID PRIMARY KEY,
    container_id VARCHAR(11) NOT NULL,
    submitted_by VARCHAR(255) NOT NULL,
    submitted_at TIMESTAMP NOT NULL,
    status VARCHAR(20) NOT NULL,
    priority VARCHAR(20) NOT NULL,
    documentation JSONB NOT NULL,
    manifest JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_verification_requests_status ON verification_requests(status);
CREATE INDEX idx_verification_requests_container_id ON verification_requests(container_id);
```

**Connection Pooling (RDS Proxy)**:
```
Max Connections: 1000
Connection Borrow Timeout: 120 seconds
Idle Client Timeout: 1800 seconds
Init Query: SET timezone='UTC'
```

**Auto-Scaling Configuration**:
```yaml
Target Metric: CPUUtilization
Target Value: 70%
Scale-in Cooldown: 300 seconds
Scale-out Cooldown: 60 seconds
Min Capacity: 2 readers
Max Capacity: 15 readers
```

**Backup Strategy**:
- Automated daily snapshots at 03:00 UTC
- Continuous backup to S3 (point-in-time recovery)
- Cross-region snapshot copy to DR region
- Snapshot retention: 35 days



### 3.6 Amazon S3 (Document Storage)

**Bucket Configuration**:
```
Primary Bucket: container-docs-primary-us-east-1
DR Bucket: container-docs-replica-eu-west-1

Versioning: Enabled
Encryption: SSE-KMS (customer-managed key)
Object Lock: Enabled (Compliance mode, 7 years)
Lifecycle Policy: Intelligent-Tiering after 30 days
Cross-Region Replication: Enabled (15-minute RTO)
```

**Bucket Structure**:
```
s3://container-docs-primary-us-east-1/
├── documents/
│   ├── {containerId}/
│   │   ├── bill-of-lading.pdf
│   │   ├── customs-declaration.pdf
│   │   ├── safety-certificate.pdf
│   │   └── packing-list.pdf
├── certifications/
│   └── {certificationId}.pdf
└── audit-exports/
    └── {year}/{month}/{day}/
```

**Access Patterns**:

1. **Document Upload**:
   - User uploads via pre-signed POST URL (15-minute expiration)
   - Server-side encryption applied automatically
   - Metadata tags: containerId, documentType, uploadedBy, uploadedAt

2. **Document Retrieval**:
   - Verification service fetches using pre-signed GET URL
   - CloudFront CDN for frequently accessed documents
   - S3 Transfer Ac"Id": "ExpireOldVersions",
      "Status": "Enabled",
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90
      }
    }
  ]
}
```

**Cross-Region Replication**:
```json
{
  "Role": "arn:aws:iam::account-id:role/s3-replication-role",
  "Rules": [
    {
      "Status": "Enabled",
      "Priority": 1,
      "Filter": {},
      "Destination": {
        "Bucket": "arn:aws:s3:::container-docs-replica-eu-west-1",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": {
            "Minutes": 15
          }
        },
        "Metrics": {
          "Status": "Enabled"
        }
      }
    }
  ]
}
```

### 3.7 Starburst Enterprise (Data Federation)

**Purpose**: Enable cross-region query federation and analytics across distributed data sources

**Deployment**:
```
Deployment: Amazon EKS (separate cluster)
Coordinator: 1 instance (r5.2xlarge)
Workers: 3-10 instances (r5.xlarge, auto-scaling)
Storage: S3 for query results and metadata
```

**Data Sources**:
- Aurora PostgreSQL (Primary region)
- Aurora PostgreSQL (DR region)
- S3 (Document metadata)
- CloudWatch Logs (via Athena connector)

**Use Cases**:

1. **Cross-Region Analytics**:
   - Query certifications across both regions
   - Aggregate statistics for reporting
   - Compliance auditing

2. **Data Lake Queries**:
   - Query S3 document metadata
   - Join with certification data
   - Generate insights on processing times

3. **Real-Time Dashboards**:
   - Live metrics from multiple sources
   - Port-specific performance analytics
   - Trend analysis

**Sample Query**:
```sql
-- Query certifications across both regions
SELECT 
    r.region,
    COUNT(*) as total_certifications,
    AVG(EXTRACT(EPOCH FROM (c.issued_at - vr.submitted_at))) as avg_processing_time_seconds
FROM 
    aurora_primary.certifications c
    JOIN aurora_primary.verification_requests vr ON c.request_id = vr.request_id
    CROSS JOIN (SELECT 'us-east-1' as region) r
UNION ALL
SELECT 
    r.region,
    COUNT(*) as total_certifications,
    AVG(EXTRACT(EPOCH FROM (c.issued_at - vr.submitted_at))) as avg_processing_time_seconds
FROM 
    aurora_dr.certifications c
    JOIN aurora_dr.verification_requests vr ON c.request_id = vr.request_id
    CROSS JOIN (SELECT 'eu-west-1' as region) r
GROUP BY r.region;
```

## 4. Security Architecture

### 4.1 Identity and Access Management (IAM)

**IAM Roles**:

```
EKS Node Role:
  - AmazonEKSWorkerNodePolicy
  - AmazonEC2ContainerRegistryReadOnly
  - AmazonEKS_CNI_Policy
  - CloudWatchAgentServerPolicy

API Gateway Service Account Role (IRSA):
  - Read from Secrets Manager (JWT signing keys)
  - Write to CloudWatch Logs
  - Publish to SNS (notifications)

Verification Service Account Role (IRSA):
  - Read/Write to Amazon MQ
  - Read from Aurora (via RDS Proxy)
  - Read from S3 (container-docs-* buckets)
  - Write to CloudWatch Logs

Certification Service Account Role (IRSA):
  - Read/Write to Aurora (via RDS Proxy)
  - Write to S3 (certifications/)
  - Use KMS (for digital signatures)
  - Write to CloudWatch Logs

Aurora RDS Role:
  - Write to S3 (backups)
  - Publish to SNS (alerts)

Lambda Execution Role (automation):
  - Manage EKS resources
  - Trigger failover procedures
  - Read CloudWatch metrics
  - Update Route 53 records
```

**Service Accounts (IRSA)**:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: verification-service-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/verification-service-role
```

### 4.2 Encryption

**Data at Rest**:
```
Aurora PostgreSQL: AES-256 (AWS KMS customer-managed key)
S3 Buckets: SSE-KMS (customer-managed key)
EBS Volumes: AES-256 (AWS KMS)
Amazon MQ: AES-256 (AWS-managed key)
Secrets Manager: AES-256 (AWS KMS customer-managed key)
CloudWatch Logs: AES-256 (AWS-managed key)
```

**Data in Transit**:
```
External Traffic: TLS 1.3 (enforced at NLB)
Internal Pod-to-Pod: mTLS (Istio service mesh)
Aurora Connections: TLS 1.2+ (enforced)
Amazon MQ Connections: AMQPS (TLS 1.2+)
S3 Access: HTTPS only (bucket policy enforced)
```

**Key Management**:
```
Service: AWS KMS
Key Type: Customer-managed keys (CMK)
Key Rotation: Automatic (annual)
Key Policy: Least privilege access
Cross-Region Keys: Replicated to DR region
```

### 4.3 Authentication and Authorization

**User Authentication**:
```
Service: AWS Cognito
User Pool: container-verification-users
MFA: Required for certification authority roles
Token Type: JWT
Access Token Expiration: 1 hour
Refresh Token Expiration: 30 days
Password Policy: 
  - Minimum 12 characters
  - Uppercase, lowercase, numbers, symbols required
  - Password history: 5
```

**API Authorization**:
```
Roles:
  - Viewer: Read-only access to certifications
  - Operator: Submit verification requests, view status
  - Certification Authority: Issue and revoke certifications
  - Administrator: Full system access, audit log access

Permissions Matrix:
  POST /api/v1/verification/submit: Operator, Certification Authority, Administrator
  GET /api/v1/certification/{id}: All authenticated users
  POST /api/v1/certification/{id}/revoke: Certification Authority, Administrator
  GET /api/v1/audit/logs: Administrator only
```

**Service-to-Service Authentication**:
```
Method: mTLS (mutual TLS)
Certificate Authority: AWS Certificate Manager (ACM)
Certificate Rotation: Automatic (90 days)
Certificate Validation: Strict (reject invalid certificates)
```

### 4.4 Network Security

**Network ACLs**:
```
Public Subnet NACL:
  Inbound:
    - Allow 443 from 0.0.0.0/0
    - Allow 80 from 0.0.0.0/0
    - Allow ephemeral ports from 0.0.0.0/0
  Outbound:
    - Allow all to 0.0.0.0/0

Private Subnet NACL:
  Inbound:
    - Allow all from VPC CIDR
  Outbound:
    - Allow all to VPC CIDR
    - Allow 443 to 0.0.0.0/0 (for AWS services)
```

**AWS WAF (Web Application Firewall)**:
```
Attached to: Network Load Balancer
Rules:
  - Rate limiting: 1000 requests per 5 minutes per IP
  - Geo-blocking: Block requests from sanctioned countries
  - SQL injection protection
  - XSS protection
  - Known bad inputs blocking
```

**AWS Shield**:
```
Service: AWS Shield Standard (automatic)
DDoS Protection: Layer 3/4 attacks
Additional: AWS Shield Advanced (optional for enhanced protection)
```

### 4.5 Compliance and Audit

**Compliance Standards**:
- SOC 2 Type II
- ISO 27001
- GDPR (for EU operations)
- PCI DSS Level 1 (if payment processing added)
- HIPAA (if health-related cargo)

**Audit Logging**:
```
AWS CloudTrail:
  - All API calls logged
  - Log file integrity validation enabled
  - Multi-region trail enabled
  - Logs stored in S3 with Object Lock

VPC Flow Logs:
  - All network traffic logged
  - Stored in CloudWatch Logs
  - Retention: 90 days

Aurora Audit Logs:
  - All database queries logged
  - Stored in CloudWatch Logs
  - Retention: 365 days

Application Audit Logs:
  - All user actions logged
  - Stored in Aurora + CloudWatch Logs
  - Retention: 7 years (compliance requirement)
```

**Security Monitoring**:
```
AWS GuardDuty:
  - Threat detection for AWS accounts
  - Monitors CloudTrail, VPC Flow Logs, DNS logs
  - Alerts on suspicious activity

AWS Security Hub:
  - Centralized security posture management
  - Aggregates findings from GuardDuty, Inspector, Macie
  - Compliance checks against CIS benchmarks

AWS Config:
  - Configuration compliance monitoring
  - Rules for encryption, public access, security groups
  - Automatic remediation for non-compliant resources

Amazon Macie:
  - Sensitive data discovery in S3
  - PII detection and classification
  - Alerts on unauthorized access to sensitive data
```



## 5. Scalability and Performance

### 5.1 Auto-Scaling Strategy

#### EKS Pod Auto-Scaling (HPA)

**Verifiow: 60 seconds
  - Max Scale-up: 100% of current pods per minute
  
Scale-down Behavior:
  - Stabilization Window: 300 seconds
  - Max Scale-down: 1 pod per minute
```

**API Gateway**:
```yaml
Min Replicas: 3
Max Replicas: 10
Target CPU: 70%
Target Memory: 80%
```

#### EKS Cluster Auto-Scaling

**Node Scaling**:
```
On-Demand Node Group:
  Min: 3 nodes
  Max: 10 nodes
  Scale-up: When pods pending for > 30 seconds
  Scale-down: When node utilization < 50% for > 10 minutes

Spot Node Group:
  Min: 0 nodes
  Max: 20 nodes
  Scale-up: When on-demand nodes at capacity
  Scale-down: When node utilization < 40% for > 5 minutes
  Interruption Handling: Graceful pod eviction
```

#### Aurora Auto-Scaling

**Read Replica Scaling**:
```
Min Replicas: 2
Max Replicas: 15
Target Metric: CPU Utilization
Target Value: 70%
Scale-in Cooldown: 300 seconds
Scale-out Cooldown: 60 seconds
```

**Storage Scaling**:
```
Initial: 10 GB
Increment: 10 GB
Maximum: 128 TB
Trigger: When storage utilization > 90%
Downtime: None (automatic)
```

### 5.2 Performance Optimization

#### Caching Strategy

**Application-Level Caching (Redis)**:
```
Deployment: Amazon ElastiCache for Redis
Instance Type: cache.r6g.large (2 vCPU, 13.07 GB RAM)
Cluster Mode: Enabled (3 shards, 2 replicas per shard)
Encryption: In-transit and at-rest

Cached Data:
  - User authentication tokens (TTL: 1 hour)
  - Frequently accessed certifications (TTL: 15 minutes)
  - Rate limit counters (TTL: 5 minutes)
  - Document metadata (TTL: 30 minutes)
```

**CDN (CloudFront)**:
```
Distribution: Global edge locations
Origin: S3 bucket (certifications)
Cache Behavior:
  - TTL: 24 hours for certification PDFs
  - Query string forwarding: Disabled
  - Compression: Enabled (gzip, brotli)
  
Security:
  - Origin Access Identity (OAI) for S3
  - Signed URLs for private documents
  - TLS 1.2+ only
```

#### Database Optimization

**Connection Pooling (RDS Proxy)**:
```
Max Connections: 1000
Connection Borrow Timeout: 120 seconds
Idle Client Timeout: 1800 seconds
Connection Reuse: Enabled
Multiplexing: Enabled
```

**Query Optimization**:
```sql
-- Indexes for common queries
CREATE INDEX idx_certifications_container_id ON certifications(container_id);
CREATE INDEX idx_certifications_status_issued_at ON certifications(status, issued_at);
CREATE INDEX idx_audit_logs_timestamp_event_type ON audit_logs(timestamp, event_type);

-- Partitioning for large tables
CREATE TABLE audit_logs_2026_02 PARTITION OF audit_logs
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

**Read/Write Splitting**:
```
Write Operations: Route to writer instance
Read Operations: Route to reader endpoint (automatic load balancing)
Consistency: Strong for writes, eventual for reads (< 100ms lag)
```

#### Network Optimization

**VPC Endpoints**:
```
S3 Gateway Endpoint: No data transfer charges
Interface Endpoints:
  - Secrets Manager
  - CloudWatch Logs
  - ECR
  - KMS
  
Benefit: Reduced latency, no internet gateway traversal
```

**Enhanced Networking**:
```
EKS Worker Nodes: Enhanced networking enabled (SR-IOV)
Network Bandwidth: Up to 10 Gbps per instance
Latency: Sub-millisecond inter-instance latency
```

### 5.3 Performance Targets

**Response Time SLAs**:
```
API Gateway:
  - P50: < 50ms
  - P95: < 100ms
  - P99: < 200ms

Verification Processing:
  - P50: < 2 seconds
  - P95: < 5 seconds
  - P99: < 10 seconds

Certification Generation:
  - P50: < 500ms
  - P95: < 1 second
  - P99: < 2 seconds

End-to-End (Submit to Certification):
  - P50: < 3 seconds
  - P95: < 8 seconds
  - P99: < 15 seconds
```

**Throughput Targets**:
```
Normal Load: 1,000 requests/minute
Peak Load: 10,000 requests/minute
Burst Capacity: 15,000 requests/minute (5 minutes)
```

**Availability SLA**:
```
System Availability: 99.99% (52 minutes downtime/year)
Regional Availability: 99.95% per region
Database Availability: 99.99% (Aurora SLA)
```

## 6. Disaster Recovery and High Availability

### 6.1 Multi-Region Failover

**RTO (Recovery Time Objective)**: 5 minutes  
**RPO (Recovery Point Objective)**: 1 minute

**Failover Architecture**:
```
Primary Region: us-east-1 (Active)
DR Region: eu-west-1 (Passive, warm standby)

Replication:
  - Aurora: Asynchronous cross-region replication (< 1 second lag)
  - S3: Cross-region replication (< 15 minutes)
  - Amazon MQ: Manual failover (standby broker in DR region)
```

**Failover Triggers**:
1. Primary region unavailability (Route 53 health checks fail)
2. Database replication lag > 10 seconds
3. Error rate > 50% for > 5 minutes
4. Manual failover initiated by operations team

**Automated Failover Process**:

```
Step 1: Detection (30 seconds)
  - Route 53 health checks fail for primary region NLB
  - CloudWatch alarms trigger SNS notifications
  - Lambda function invoked for failover orchestration

Step 2: DNS Failover (60 seconds)
  - Route 53 updates DNS to point to DR region NLB
  - TTL: 60 seconds for fast propagation
  - Health check interval: 30 seconds

Step 3: Database Promotion (120 seconds)
  - Promote Aurora read replica in DR region to writer
  - Update RDS Proxy endpoint configuration
  - Update application connection strings via Secrets Manager

Step 4: Service Activation (120 seconds)
  - Scale up EKS pods in DR region (from 3 to 10 for each service)
  - Activate Amazon MQ broker in DR region
  - Update application configuration to use DR resources

Step 5: Verification (60 seconds)
  - Run smoke tests to verify system functionality
  - Monitor error rates and latency
  - Confirm all services operational
  - Notify operations team of successful failover

Total Failover Time: ~5 minutes
```

**Failback Process**:
```
1. Verify primary region is fully operational
2. Synchronize data from DR region to primary region
3. Update Route 53 to use weighted routing (10% primary, 90% DR)
4. Gradually shift traffic: 25%, 50%, 75%, 100% to primary
5. Monitor for errors during traffic shift
6. Scale down DR region resources to warm standby
7. Verify replication from primary to DR is working
```

### 6.2 High Availability Within Region

**Multi-AZ Deployment**:
```
EKS Worker Nodes: Distributed across 2 availability zones
Aurora: Writer in AZ-1, Readers in AZ-1 and AZ-2
Amazon MQ: Active in AZ-1, Standby in AZ-2
NAT Gateways: One per availability zone
Network Load Balancer: Cross-zone load balancing enabled
```

**Pod Disruption Budgets**:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: verification-service-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: verification-service
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: certification-service-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: certification-service
```

**Health Checks**:
```
NLB Health Checks:
  - Protocol: HTTP
  - Path: /health
  - Interval: 30 seconds
  - Timeout: 10 seconds
  - Healthy Threshold: 2
  - Unhealthy Threshold: 2

Kubernetes Liveness Probes:
  - Initial Delay: 30 seconds
  - Period: 10 seconds
  - Timeout: 5 seconds
  - Failure Threshold: 3

Kubernetes Readiness Probes:
  - Initial Delay: 10 seconds
  - Period: 5 seconds
  - Timeout: 3 seconds
  - Failure Threshold: 3
```

### 6.3 Backup and Recovery

**Database Backups**:
```
Automated Snapshots:
  - Frequency: Daily at 03:00 UTC
  - Retention: 35 days
  - Cross-region copy: Yes (to DR region)

Continuous Backup:
  - Point-in-time recovery: Up to 35 days
  - Backup to S3: Automatic
  - Encryption: AES-256 (KMS)

Manual Snapshots:
  - Before major changes
  - Retention: Indefinite (until manually deleted)
```

**S3 Backup**:
```
Versioning: Enabled (all objects)
Cross-Region Replication: Enabled (15-minute RTO)
Object Lock: Compliance mode (7 years)
Lifecycle Policy: Transition to Glacier after 1 year
```

**Configuration Backup**:
```
Infrastructure as Code: Terraform (stored in Git)
EKS Manifests: Stored in Git (GitOps with ArgoCD)
Secrets: AWS Secrets Manager (cross-region replication)
AMI Backups: Automated weekly snapshots
```

**Recovery Procedures**:

1. **Database Recovery**:
   ```
   Point-in-Time Recovery:
     - Restore to specific timestamp
     - Create new cluster from snapshot
     - Update application connection strings
     - Verify data integrity
     - Switch traffic to new cluster
   ```

2. **S3 Object Recovery**:
   ```
   Version Recovery:
     - List object versions
     - Restore specific version
     - Verify object integrity
   ```

3. **Complete Region Failure**:
   ```
   - Execute automated failover to DR region
   - Promote DR Aurora to writer
   - Scale up DR EKS cluster
   - Update Route 53 DNS
   - Verify all services operational
   ```



## 7. Monitoring and Observability

### 7.1 Metrics Collection

**Amazon CloudWatch Metrics**:

```
EKS Metrics (Container Insights):
  - Pod CPU utilization
  - Pod memory utilization
  - Pod network I/O
  - Container restart count
  - Node CPU/memory utilization
  - Cluster-level metrics

Aurora Metrics (Performance Insights):
  - Database connections
  - CPU utilization
  - Read/write IOPS
  - Query latency (P50, P95, P99)
  - Replication lag
  - Deadlocks and slow queries

Amazon MQ Metrics:
  - Queue depth
  - Message enqueue rate
  - Message dequeue rate
  - Consumer count
  - Dead letter queue depth
  - Broker CPU/memory utilization

Application Metrics (Custom):
  - Request rate (requests/minute)
  - Error rate (errors/total requests)
  - Response time (P50, P95, P99)
  - Verification success rate
  - Certification generation time
  - Queue processing time
```

**Metric Aggregation**:
```
Collection Interval: 1 minute
Retention:
  - High-resolution (1-minute): 15 days
  - Standard (5-minute): 63 days
  - Aggregated (1-hour): 455 days
```

### 7.2 Logging

**Log Aggregation (CloudWatch Logs)**:

```
Log Groups:
  - /aws/eks/container-verification/api-gateway
  - /aws/eks/container-verification/verification-service
  - /aws/eks/container-verification/certification-service
  - /aws/rds/aurora/container-verification
  - /aws/amazonmq/container-verification
  - /aws/lambda/failover-orchestration

Log Format: JSON structured logging
Log Level: INFO (production), DEBUG (development)
Retention: 90 days (application logs), 365 days (audit logs)
```

**Log Structure**:
```json
{
  "timestamp": "2026-02-14T10:30:00.123Z",
  "level": "INFO",
  "service": "verification-service",
  "traceId": "1-5f8a1234-abcdef1234567890",
  "spanId": "abcdef123456",
  "userId": "user-123",
  "requestId": "req-456",
  "containerId": "ABCU1234567",
  "message": "Verification completed successfully",
  "duration_ms": 2345,
  "status": "PASSED"
}
```

**Log Insights Queries**:
```
-- Error rate by service
fields @timestamp, service, message
| filter level = "ERROR"
| stats count() by service
| sort count desc

-- Slow requests (> 5 seconds)
fields @timestamp, requestId, duration_ms, service
| filter duration_ms > 5000
| sort duration_ms desc

-- Verification failure reasons
fields @timestamp, containerId, failureReason
| filter status = "FAILED"
| stats count() by failureReason
```

### 7.3 Distributed Tracing

**AWS X-Ray Configuration**:

```
Sampling Rate: 10% of requests (to reduce costs)
Trace Retention: 30 days

Instrumented Services:
  - API Gateway (all requests)
  - Verification Service (all processing)
  - Certification Service (all operations)
  - Aurora queries (via RDS Proxy)
  - S3 operations
  - Amazon MQ operations

Trace Segments:
  - HTTP request/response
  - Database queries
  - External API calls
  - Queue operations
  - S3 operations
```

**Trace Example**:
```
Request: POST /api/v1/verification/submit
├─ API Gateway (50ms)
│  ├─ Authentication (10ms)
│  ├─ Validation (5ms)
│  └─ Enqueue to MQ (35ms)
├─ Verification Service (2.3s)
│  ├─ Dequeue from MQ (10ms)
│  ├─ Fetch documents from S3 (200ms)
│  ├─ Document validation (1.5s)
│  ├─ Manifest verification (500ms)
│  └─ Update status in Aurora (100ms)
└─ Certification Service (500ms)
   ├─ Generate certification (200ms)
   ├─ Sign with KMS (100ms)
   ├─ Store in Aurora (100ms)
   └─ Upload PDF to S3 (100ms)

Total Duration: 2.85s
```

### 7.4 Alerting

**CloudWatch Alarms**:

```
Critical Alarms (PagerDuty):
  - Error rate > 5% for 5 minutes
  - API Gateway 5xx errors > 10 in 1 minute
  - Aurora CPU > 90% for 5 minutes
  - Aurora replication lag > 10 seconds
  - EKS node failure (any node)
  - Amazon MQ broker failure
  - Dead letter queue depth > 100

Warning Alarms (Email/Slack):
  - Error rate > 2% for 5 minutes
  - Response time P95 > 5 seconds for 5 minutes
  - Queue depth > 1000 for 5 minutes
  - Aurora CPU > 70% for 10 minutes
  - EKS pod restart count > 5 in 10 minutes
  - S3 replication lag > 30 minutes

Informational Alarms (Slack):
  - Auto-scaling event (scale up/down)
  - Deployment completed
  - Backup completed
  - Certificate rotation completed
```

**Alarm Actions**:
```yaml
Critical:
  - Send to PagerDuty (immediate notification)
  - Execute auto-remediation Lambda (if applicable)
  - Create incident in ServiceNow

Warning:
  - Send email to operations team
  - Post to Slack #alerts channel
  - Log to incident tracking system

Informational:
  - Post to Slack #operations channel
  - Log to CloudWatch Logs
```

### 7.5 Dashboards

**CloudWatch Dashboard - System Overview**:
```
Widgets:
  - Request rate (line chart, 1-hour window)
  - Error rate (line chart, 1-hour window)
  - Response time P50/P95/P99 (line chart, 1-hour window)
  - Active EKS pods by service (stacked area chart)
  - Aurora connections (line chart)
  - Queue depth (line chart)
  - S3 request rate (line chart)
  - Cost estimate (number widget)
```

**CloudWatch Dashboard - Service Health**:
```
Widgets:
  - Service status (green/yellow/red indicators)
  - Recent errors (log insights widget)
  - Slow requests (log insights widget)
  - Database query performance (top 10 slow queries)
  - Cache hit rate (line chart)
  - Network throughput (line chart)
```

**Grafana Dashboard (Optional)**:
```
Data Sources:
  - CloudWatch
  - Prometheus (for EKS metrics)
  - Aurora Performance Insights

Dashboards:
  - Executive summary (high-level KPIs)
  - Service-level metrics (detailed per service)
  - Infrastructure metrics (EKS, Aurora, MQ)
  - Cost analysis (resource utilization vs cost)
```

## 8. Data Models

### 8.1 Core Data Models

**VerificationRequest**:
```json
{
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "containerId": "ABCU1234567",
  "submittedBy": "user@example.com",
  "submittedAt": "2026-02-14T10:30:00Z",
  "status": "PENDING",
  "priority": "STANDARD",
  "documentation": {
    "billOfLading": {
      "documentId": "BOL-123456",
      "s3Uri": "s3://container-docs/ABCU1234567/bill-of-lading.pdf",
      "issuer": "Maersk Line",
      "issuedDate": "2026-02-10T00:00:00Z",
      "expirationDate": "2026-03-10T00:00:00Z",
      "signature": "SHA256:abc123..."
    },
    "customsDeclaration": {
      "documentId": "CD-789012",
      "s3Uri": "s3://container-docs/ABCU1234567/customs-declaration.pdf",
      "issuer": "US Customs",
      "issuedDate": "2026-02-12T00:00:00Z",
      "expirationDate": "2026-03-12T00:00:00Z",
      "signature": "SHA256:def456..."
    },
    "safetyCertificate": {
      "documentId": "SC-345678",
      "s3Uri": "s3://container-docs/ABCU1234567/safety-certificate.pdf",
      "issuer": "International Maritime Organization",
      "issuedDate": "2026-01-15T00:00:00Z",
      "expirationDate": "2027-01-15T00:00:00Z",
      "signature": "SHA256:ghi789..."
    },
    "packingList": {
      "documentId": "PL-901234",
      "s3Uri": "s3://container-docs/ABCU1234567/packing-list.pdf",
      "issuer": "Shipper Inc",
      "issuedDate": "2026-02-13T00:00:00Z",
      "signature": "SHA256:jkl012..."
    }
  },
  "manifest": {
    "items": [
      {
        "description": "Electronic components",
        "quantity": 1000,
        "weight": 500.5,
        "unit": "kg",
        "hsCode": "8542.31",
        "value": 50000.00,
        "currency": "USD"
      },
      {
        "description": "Textile products",
        "quantity": 500,
        "weight": 250.0,
        "unit": "kg",
        "hsCode": "6302.60",
        "value": 15000.00,
        "currency": "USD"
      }
    ],
    "totalWeight": 750.5,
    "totalValue": 65000.00
  }
}
```

**VerificationResult**:
```json
{
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "PASSED",
  "timestamp": "2026-02-14T10:32:15Z",
  "processingTime": 2.35,
  "checks": [
    {
      "checkType": "DOCUMENT_COMPLETENESS",
      "passed": true,
      "details": "All required documents present",
      "timestamp": "2026-02-14T10:32:10Z"
    },
    {
      "checkType": "DOCUMENT_AUTHENTICITY",
      "passed": true,
      "details": "All signatures valid",
      "timestamp": "2026-02-14T10:32:12Z"
    },
    {
      "checkType": "MANIFEST_VERIFICATION",
      "passed": true,
      "details": "Contents match manifest within tolerance",
      "timestamp": "2026-02-14T10:32:14Z"
    },
    {
      "checkType": "COMPLIANCE_CHECK",
      "passed": true,
      "details": "No compliance violations found",
      "timestamp": "2026-02-14T10:32:15Z"
    }
  ],
  "failureReasons": [],
  "complianceViolations": [],
  "riskScore": 0.15
}
```

**Certification**:
```json
{
  "certificationId": "cert-550e8400-e29b-41d4-a716-446655440001",
  "containerId": "ABCU1234567",
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "issuedAt": "2026-02-14T10:32:30Z",
  "expiresAt": "2026-05-15T10:32:30Z",
  "issuedBy": "Port Authority - New York",
  "issuingOfficer": "John Doe",
  "issuingOfficerId": "officer-123",
  "verificationResult": {
    "status": "PASSED",
    "checks": [...]
  },
  "digitalSignature": "SHA256:mno345...",
  "signatureAlgorithm": "RSA-SHA256",
  "certificatePdfUri": "s3://container-docs/certifications/cert-550e8400.pdf",
  "status": "ACTIVE",
  "metadata": {
    "version": "1.0",
    "standard": "ISO 28000",
    "region": "us-east-1"
  }
}
```

**AuditLog**:
```json
{
  "logId": "log-550e8400-e29b-41d4-a716-446655440002",
  "timestamp": "2026-02-14T10:32:30Z",
  "eventType": "CERTIFICATION_ISSUED",
  "actorId": "officer-123",
  "actorName": "John Doe",
  "actorRole": "CERTIFICATION_AUTHORITY",
  "containerId": "ABCU1234567",
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "certificationId": "cert-550e8400-e29b-41d4-a716-446655440001",
  "outcome": "SUCCESS",
  "details": {
    "action": "Certification issued after successful verification",
    "ipAddress": "203.0.113.42",
    "userAgent": "Mozilla/5.0...",
    "location": "New York, USA"
  },
  "metadata": {
    "region": "us-east-1",
    "service": "certification-service",
    "version": "1.0"
  }
}
```

### 8.2 Event Types

**Verification Events**:
- `VERIFICATION_REQUEST_SUBMITTED`
- `VERIFICATION_IN_PROGRESS`
- `VERIFICATION_COMPLETED`
- `VERIFICATION_FAILED`

**Certification Events**:
- `CERTIFICATION_ISSUED`
- `CERTIFICATION_RETRIEVED`
- `CERTIFICATION_REVOKED`
- `CERTIFICATION_EXPIRED`

**Security Events**:
- `AUTHENTICATION_SUCCESS`
- `AUTHENTICATION_FAILED`
- `AUTHORIZATION_DENIED`
- `UNAUTHORIZED_ACCESS_ATTEMPT`

**System Events**:
- `SERVICE_STARTED`
- `SERVICE_STOPPED`
- `FAILOVER_INITIATED`
- `FAILOVER_COMPLETED`
- `BACKUP_COMPLETED`
- `SCALING_EVENT`

