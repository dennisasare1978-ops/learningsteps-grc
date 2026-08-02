# LearningSteps GRC Assessment — System Scope

## 1. Purpose

The purpose of this document is to define the boundaries of the Governance, Risk, and Compliance (GRC) security assessment for the LearningSteps environment.

The assessment evaluates the security posture of the deployed LearningSteps application and its supporting cloud infrastructure. The objective is to identify security risks, evaluate existing controls, identify control gaps, and recommend improvements that support the organization's security and compliance objectives.

ISO/IEC 27001 is used as the primary reference framework for control mapping.

---

## 2. System Overview

LearningSteps is a cloud-hosted learning journal application deployed in Microsoft Azure.

The application uses a two-tier architecture consisting of:

1. A FastAPI application layer running as containerized workloads in Azure Kubernetes Service (AKS).
2. A PostgreSQL database layer using Azure Database for PostgreSQL Flexible Server.

The application is deployed and managed using Infrastructure as Code and CI/CD technologies.

The current environment includes:

- FastAPI application
- Azure Kubernetes Service (AKS)
- Azure Container Registry (ACR)
- Azure Database for PostgreSQL Flexible Server
- Kubernetes Deployments, Services, Secrets, and health probes
- Azure Load Balancer
- Terraform Infrastructure as Code
- GitHub source control
- GitHub Actions CI/CD
- Application health and metrics endpoints

---

## 3. Assessment Scope

The following components are considered **in scope** for this GRC assessment.

### 3.1 Application

The LearningSteps FastAPI application is in scope.

The assessment considers:

- API exposure
- Application configuration
- Health endpoints
- Application-to-database connectivity
- Container deployment
- Application security configuration

The deployed application provides API endpoints including health checks, metrics, and learning journal entry operations.

### 3.2 Azure Kubernetes Service

The AKS environment hosting the LearningSteps API is in scope.

The deployed environment currently consists of:

- AKS cluster: `aks-west-eu`
- Two Kubernetes worker nodes
- Kubernetes version 1.35.6
- Two LearningSteps API replicas
- Kubernetes liveness and readiness probes
- Container resource requests and limits
- Container security context

AKS is responsible for orchestrating and maintaining the application containers.

### 3.3 Azure Container Registry

Azure Container Registry is in scope.

The registry:

`acrwesteu.azurecr.io`

stores the LearningSteps application container images used by AKS.

Access between AKS and ACR is controlled using Azure role-based access control with the `AcrPull` role.

### 3.4 Database

Azure Database for PostgreSQL Flexible Server is in scope.

The deployed database environment includes:

- PostgreSQL server: `psql-dev-aks-west-eu`
- PostgreSQL version: 16
- Application database: `learning_journal`

The database stores information created and retrieved through the LearningSteps API.

### 3.5 Kubernetes Configuration

Kubernetes configuration used to operate the application is in scope.

This includes:

- Deployments
- Services
- Kubernetes Secrets
- Service accounts
- Role-based access configuration
- Liveness probes
- Readiness probes
- Container security settings
- Resource limits

Database connection information is provided to the application through a Kubernetes Secret rather than being directly embedded in the Deployment manifest.

### 3.6 Network Exposure

The network path used to access LearningSteps is in scope.

The application is currently exposed using a Kubernetes Service of type `LoadBalancer`.

At the time of assessment, the application was verified as publicly reachable through the Azure-assigned Load Balancer IP address.

Network exposure, API accessibility, firewall configuration, and access restrictions will therefore be considered during the risk assessment.

### 3.7 Infrastructure as Code

Terraform configuration used to deploy the Azure environment is in scope.

Terraform manages infrastructure including:

- Azure resource group
- AKS
- ACR
- PostgreSQL
- PostgreSQL database
- PostgreSQL firewall configuration
- AKS-to-ACR permissions

Terraform state is stored remotely using Azure Blob Storage.

### 3.8 Source Control and CI/CD

The GitHub repository and GitHub Actions pipelines used to manage and deploy LearningSteps are in scope.

The CI/CD environment includes:

- Application source code
- Infrastructure configuration
- Kubernetes manifests
- Infrastructure pipeline
- Application deployment pipeline
- Container image build and push process
- Automated AKS deployment

The assessment considers risks associated with source control, credentials, pipeline permissions, and deployment automation.

### 3.9 Monitoring and Operational Security

Existing monitoring-related capabilities are in scope.

These include:

- `/health` endpoint
- `/metrics` endpoint
- Kubernetes liveness probes
- Kubernetes readiness probes
- Existing monitoring configuration stored in the technical repository

The assessment will determine whether these capabilities provide sufficient security monitoring and audit evidence.

---

## 4. Out of Scope

The following areas are outside the scope of this assessment:

- Security assessment of Microsoft Azure's underlying physical data centres
- Security of Microsoft's internal Azure platform infrastructure
- GitHub's internal infrastructure
- End-user personal devices
- Internet service provider infrastructure
- Third-party systems not directly integrated with LearningSteps
- Physical security of cloud provider facilities
- Formal penetration testing of Microsoft-managed services

These services may affect LearningSteps but are primarily controlled by their respective service providers.

---

## 5. Assessment Assumptions

The assessment is based on the following assumptions:

1. The deployed environment represents the current LearningSteps implementation.
2. Azure and GitHub managed services operate according to their documented shared-responsibility models.
3. Administrative access used during the assessment is authorized.
4. The development environment is being assessed as the current technical baseline.
5. Findings will be based on available technical evidence and configuration.
6. Controls that cannot be demonstrated with evidence will not automatically be considered effective.
7. The assessment represents a point-in-time review and the environment may change after evidence is collected.

---

## 6. Assessment Boundary

The primary assessment boundary can be represented as:

GitHub Source Control  
↓  
GitHub Actions CI/CD  
↓  
Azure Container Registry  
↓  
Azure Kubernetes Service  
↓  
LearningSteps FastAPI Application  
↓  
Azure Database for PostgreSQL

Supporting Terraform, Kubernetes configuration, secrets management, networking, access control, and monitoring mechanisms are included within this boundary.

---

## 7. Verified Technical Baseline

Before beginning the GRC assessment, the technical environment was verified as operational.

The following conditions were confirmed:

- Terraform infrastructure deployment completed successfully.
- Azure Container Registry was provisioned successfully.
- PostgreSQL Flexible Server reported a `Ready` state.
- Two AKS worker nodes reported `Ready`.
- Kubernetes version 1.35.6 was operational.
- Two LearningSteps API pods reported `Running` and `Ready`.
- The Kubernetes LoadBalancer received a public IP address.
- The `/health` endpoint returned a healthy status.
- The `/entries` endpoint successfully returned a database-backed response.
- The `/docs` endpoint returned HTTP 200.
- The GitHub infrastructure pipeline completed successfully.
- The GitHub application deployment pipeline completed successfully.

This operational baseline provides the technical foundation required for the GRC assessment.

---

## 8. Scope Statement

The LearningSteps GRC assessment covers the application, cloud infrastructure, database, container platform, deployment pipeline, infrastructure code, secrets handling, network exposure, access controls, and monitoring capabilities directly supporting the deployed LearningSteps environment.

The assessment will use this defined scope to identify assets, threats, risks, existing controls, security gaps, and prioritized improvements.
