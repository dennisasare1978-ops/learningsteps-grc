# LearningSteps GRC Assessment — Asset Inventory

## 1. Purpose

The purpose of this asset inventory is to identify and classify the information, application, infrastructure, credential, development, and operational assets that support the LearningSteps environment.

Asset identification is an important part of risk management because an organization must understand what it owns and operates before it can determine what needs protection.

Each asset is evaluated according to:

- Asset category
- Asset owner
- Sensitivity
- Business importance
- Security considerations

---

## 2. Asset Classification

For this assessment, assets are classified using three sensitivity levels:

### Low

Compromise or loss would have limited security or business impact.

### Medium

Compromise, modification, or loss could affect normal operations or expose internal technical information.

### High

Compromise, unauthorized disclosure, modification, or loss could significantly affect the confidentiality, integrity, or availability of LearningSteps.

Business importance is also classified as Low, Medium, or High based on how important the asset is to continued operation of the service.

---

## 3. LearningSteps Asset Inventory

| ID | Asset | Category | Owner | Sensitivity | Business Importance | Reason |
|---|---|---|---|---|---|---|
| A-01 | LearningSteps FastAPI application | Application | Application Owner | Medium | High | Provides the core API functionality used by LearningSteps. |
| A-02 | Learning journal data | Data | Application/Data Owner | High | Represents application information stored and processed by LearningSteps. Unauthorized disclosure or modification could affect users and data integrity. |
| A-03 | PostgreSQL Flexible Server | Database | Cloud/Platform Owner | High | Stores persistent LearningSteps application data and is required for database-backed API operations. |
| A-04 | `learning_journal` database | Data/Database | Application/Data Owner | High | Contains persistent application records and is essential to application functionality. |
| A-05 | PostgreSQL administrator credentials | Credential | Cloud/Platform Owner | High | Administrative credentials could provide privileged access to the database if compromised. |
| A-06 | Kubernetes `learningsteps-secrets` Secret | Credential/Secret | Cloud/Platform Owner | High | Contains the database connection information required by the application. |
| A-07 | AKS cluster `aks-west-eu` | Compute/Platform | Cloud/Platform Owner | High | Hosts and orchestrates the LearningSteps application workloads. |
| A-08 | AKS worker nodes | Compute | Cloud/Platform Owner | High | Provide the compute capacity on which the application containers execute. |
| A-09 | LearningSteps Kubernetes Deployment | Configuration/Workload | DevOps/Platform Owner | Medium | Defines how application replicas, resources, probes, environment variables, and security settings are deployed. |
| A-10 | Azure Load Balancer / public application endpoint | Network | Cloud/Platform Owner | High | Provides external access to LearningSteps and forms part of the application's public attack surface. |
| A-11 | Azure Container Registry `acrwesteu` | Container Registry | DevOps/Platform Owner | High | Stores container images deployed to the AKS environment. |
| A-12 | LearningSteps container images | Software Artifact | DevOps/Application Owner | High | Executable application artifacts deployed into production workloads. Tampering could affect the integrity of the application. |
| A-13 | Terraform configuration | Infrastructure as Code | Cloud/DevOps Owner | High | Defines significant parts of the Azure infrastructure and security configuration. |
| A-14 | Terraform remote state | Configuration/Sensitive Metadata | Cloud/DevOps Owner | High | Contains infrastructure state and metadata used by Terraform to manage Azure resources. |
| A-15 | GitHub `learningsteps-k8s-final` repository | Source Code | Development/DevOps Owner | High | Contains application code, Terraform configuration, Kubernetes manifests, and CI/CD definitions. |
| A-16 | GitHub Actions workflows | CI/CD | DevOps Owner | High | Automate infrastructure and application deployment and therefore have the ability to influence the deployed environment. |
| A-17 | CI/CD credentials and Azure authentication material | Credential | DevOps/Cloud Owner | High | Could permit unauthorized Azure or deployment operations if exposed or misused. |
| A-18 | Kubernetes RBAC configuration | Access Control | Cloud/Platform Owner | High | Determines permissions granted to Kubernetes identities and deployment processes. |
| A-19 | Health and metrics endpoints | Monitoring | Application/Operations Owner | Medium | Provide operational information used to determine application health and observability. |
| A-20 | GRC assessment evidence | Governance/Audit | Security/GRC Owner | Medium | Provides evidence supporting security findings, control evaluations, and audit conclusions. |

---

## 4. Critical Assets

The following assets are considered particularly important to the security of LearningSteps:

- Application and learning journal data
- PostgreSQL database and administrative credentials
- Kubernetes secrets
- AKS cluster
- Container images and Azure Container Registry
- Terraform configuration and remote state
- GitHub source repository
- GitHub Actions deployment pipeline
- CI/CD and Azure authentication credentials

Compromise of these assets could allow an attacker to access sensitive information, modify the application, disrupt availability, alter infrastructure, or gain unauthorized access to cloud resources.

---

## 5. Asset Ownership

Asset ownership identifies who is responsible for ensuring that an asset is appropriately managed and protected.

For this assessment, ownership is expressed using functional roles rather than individual names.

Examples include:

- **Application Owner** — responsible for application functionality and application-level security.
- **Data Owner** — responsible for determining appropriate protection of application information.
- **Cloud/Platform Owner** — responsible for Azure and Kubernetes infrastructure.
- **DevOps Owner** — responsible for deployment automation, container delivery, and infrastructure pipelines.
- **Security/GRC Owner** — responsible for security governance, risk assessment, compliance evidence, and policy oversight.

In a larger organization, these functional roles would normally be assigned to named individuals or teams.

---

## 6. Asset Inventory Observations

The inventory demonstrates that LearningSteps depends on more than the FastAPI application and PostgreSQL database.

The security of the system also depends on the integrity of the software supply chain, infrastructure code, container registry, Kubernetes configuration, deployment credentials, and CI/CD pipeline.

Several assets have been classified as High sensitivity because compromise could provide privileged access or allow significant changes to the deployed environment.

These assets will receive particular attention during the threat modeling and risk assessment stages.
