# LearningSteps GRC Assessment — Threat Model

## 1. Purpose

This threat model identifies realistic security threats that could affect the LearningSteps environment.

The analysis is based on the actual deployed architecture and the assets identified in the LearningSteps asset inventory.

The objective is not to identify every theoretically possible attack. Instead, the assessment focuses on threats that are relevant to the application's Azure, Kubernetes, database, source-control, and CI/CD architecture.

---

## 2. Threat Sources

Four primary threat sources are considered.

### External Attacker

An individual or group outside the organization attempting to gain unauthorized access, disrupt services, steal information, or exploit publicly accessible components.

### Malicious or Unauthorized User

A user who has legitimate or partial access to the application but attempts to access or modify information beyond their authorization.

### Misconfiguration or Human Error

Security weaknesses introduced accidentally through cloud configuration, Kubernetes configuration, networking, credentials, CI/CD, or infrastructure code.

### Insider or Credential Compromise

A trusted account, administrator, developer, or compromised credential that provides access to sensitive systems or deployment processes.

---

## 3. Architecture Attack Surface

The main LearningSteps attack surface includes:

- Public FastAPI endpoints
- Azure Load Balancer
- AKS cluster
- Kubernetes workloads
- Kubernetes Secrets
- PostgreSQL database
- Azure Container Registry
- Container images
- GitHub repository
- GitHub Actions
- Terraform configuration and state
- Azure administrative identities and credentials
- Monitoring and metrics endpoints

The public application endpoint represents the most direct external entry point, while GitHub, CI/CD credentials, Kubernetes secrets, and Azure administrative access represent important privileged attack paths.

---

## 4. Identified Threats

| ID | Threat | Threat Source | Target Asset | Potential Consequence |
|---|---|---|---|---|
| T-01 | Unauthorized access to publicly exposed API endpoints | External attacker | FastAPI / Load Balancer | Unauthorized access, data exposure, modification, or abuse of application functionality |
| T-02 | Exploitation of application vulnerabilities | External attacker | FastAPI application | Application compromise, data access, service disruption, or further access into the environment |
| T-03 | Excessive public network exposure | Misconfiguration | Load Balancer / AKS / API | Increased attack surface and unauthorized access attempts |
| T-04 | PostgreSQL firewall or network misconfiguration | Misconfiguration | PostgreSQL | Unauthorized database connectivity or exposure |
| T-05 | Database credential compromise | External attacker / Insider | PostgreSQL credentials | Unauthorized reading, modification, or deletion of application data |
| T-06 | Kubernetes Secret exposure | Insider / Credential compromise | `learningsteps-secrets` | Disclosure of database connection credentials |
| T-07 | Secrets accidentally committed to GitHub | Human error | GitHub repository / Credentials | Credential disclosure and unauthorized cloud, database, or deployment access |
| T-08 | Compromise of GitHub account or repository access | Credential compromise | GitHub repository | Unauthorized source-code, infrastructure, or pipeline modification |
| T-09 | Compromise of GitHub Actions credentials | Credential compromise | CI/CD / Azure | Unauthorized deployment or modification of Azure resources |
| T-10 | Malicious modification of CI/CD workflow | Insider / Compromised account | GitHub Actions | Malicious code deployment, credential theft, or infrastructure changes |
| T-11 | Malicious or vulnerable container image | Supply-chain attacker / Human error | ACR / AKS | Execution of vulnerable or malicious software in the cluster |
| T-12 | Unpatched container dependencies | Human error / Operational weakness | FastAPI container | Exploitation of known vulnerabilities |
| T-13 | Excessive Kubernetes RBAC permissions | Misconfiguration | AKS / Kubernetes RBAC | Unauthorized modification or access to cluster resources |
| T-14 | Compromise of AKS administrative access | Credential compromise | AKS | Cluster-wide control, workload modification, or secret access |
| T-15 | Terraform state exposure | Human error / Credential compromise | Terraform state | Disclosure of infrastructure metadata and potentially sensitive configuration |
| T-16 | Unauthorized Terraform changes | Insider / Compromised CI/CD | Azure infrastructure | Security controls could be weakened or infrastructure modified |
| T-17 | Insufficient logging and security monitoring | Operational weakness | Entire environment | Security incidents may not be detected or investigated promptly |
| T-18 | Denial-of-service against the public API | External attacker | Load Balancer / FastAPI / AKS | Application degradation or loss of availability |
| T-19 | Unauthorized modification or deletion of learning data | External attacker / Unauthorized user | PostgreSQL / learning data | Loss of integrity and potential business impact |
| T-20 | Insufficient backup or recovery capability | Operational weakness | PostgreSQL / Application data | Permanent or prolonged data loss following failure or attack |
| T-21 | Public exposure of operational endpoints | External attacker | `/metrics`, `/health`, `/docs` | Disclosure of technical information useful for reconnaissance |
| T-22 | Dependency or software supply-chain compromise | Supply-chain attacker | Application dependencies / container build | Malicious code could enter the application through trusted software components |

---

## 5. Key Threat Scenarios

### Scenario 1 — Public API Attack

An external attacker discovers the publicly accessible LearningSteps endpoint and probes the API for vulnerabilities.

If authentication, authorization, input validation, or application protections are insufficient, the attacker may be able to access information, modify records, or disrupt the service.

### Scenario 2 — Credential Leakage

A database, Azure, Kubernetes, or CI/CD credential is accidentally committed to source control or otherwise exposed.

An attacker obtaining the credential could potentially access sensitive systems without needing to exploit the application itself.

### Scenario 3 — CI/CD Pipeline Compromise

An attacker compromises an account or credential capable of modifying GitHub Actions.

Because the deployment pipeline can build container images and deploy to AKS, compromise of the pipeline could allow malicious software to be introduced into the environment.

### Scenario 4 — Container Supply-Chain Attack

A vulnerable or malicious dependency becomes part of the FastAPI container image.

The image is stored in ACR and subsequently deployed to AKS through the trusted CI/CD process.

This demonstrates that automation alone does not guarantee software integrity.

### Scenario 5 — Database Compromise

An attacker obtains database credentials or takes advantage of an overly permissive network configuration.

This could allow unauthorized access to LearningSteps data, resulting in confidentiality, integrity, and availability impacts.

### Scenario 6 — Insufficient Detection

An attacker gains access or repeatedly probes the environment, but insufficient centralized logging or alerting prevents timely detection.

The technical environment may continue operating while the security incident remains unnoticed.

---

## 6. Existing Security Features Observed

The technical assessment has already identified several existing security features, including:

- Kubernetes Secrets used instead of embedding the database password directly in the Deployment manifest
- AKS-to-ACR access using the `AcrPull` role
- Container configured to run as a non-root user
- Linux container capabilities dropped
- Privilege escalation disabled
- CPU and memory resource limits
- Kubernetes liveness and readiness probes
- Terraform Infrastructure as Code
- GitHub Actions deployment automation
- PostgreSQL TLS requested by the application's database connection
- Application health and metrics capabilities

These controls may reduce some threats, but their effectiveness will be evaluated later during control mapping and gap analysis.

---

## 7. Threat Model Conclusion

The LearningSteps environment has multiple realistic attack paths because it combines a publicly accessible API, cloud infrastructure, containerized workloads, persistent data, privileged credentials, and automated deployment pipelines.

The highest areas of concern are expected to involve:

- Public application exposure
- Identity and credential security
- Database protection
- CI/CD security
- Kubernetes access control
- Container and dependency security
- Logging and incident detection
- Backup and recovery

The identified threats will now be evaluated through a formal risk assessment using likelihood and impact ratings.
