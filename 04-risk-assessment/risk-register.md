# LearningSteps Security Risk Assessment

## 1. Purpose

This document records the security risks identified during the evidence-based security assessment of the LearningSteps environment.

The risk assessment is based on technical evidence collected directly from the Azure subscription, Azure Kubernetes Service (AKS), Azure Database for PostgreSQL Flexible Server, Azure Container Registry (ACR), Kubernetes workloads, Terraform configuration, GitHub Actions workflows, and the LearningSteps application repository.

This assessment is a security configuration and governance assessment. It is not a penetration test.

No exploitation was performed.

---

## 2. Assessment Approach

The assessment followed the sequence:

1. Define system scope.
2. Identify assets.
3. Collect technical security evidence.
4. Identify threats and weaknesses.
5. Identify existing controls.
6. Determine likelihood based on observed conditions.
7. Determine business/security impact.
8. Calculate risk.
9. Map risks to ISO/IEC 27001:2022 controls.
10. Develop security improvement recommendations.

Likelihood ratings were not assigned until technical evidence had been collected.

---

# 3. Risk Rating Methodology

## 3.1 Likelihood

Likelihood represents the probability that the identified threat scenario could occur given the observed exposure, weaknesses, existing controls, and attacker opportunity.

| Score | Rating | Description |
|---|---|---|
| 1 | Rare | Strong controls make exploitation or occurrence unlikely. |
| 2 | Unlikely | Some exposure exists, but effective controls significantly reduce probability. |
| 3 | Possible | Realistic conditions exist for the threat to occur. |
| 4 | Likely | Significant exposure or control weaknesses make occurrence reasonably probable. |
| 5 | Almost Certain | The condition is highly exposed, easily exploitable, or expected to occur frequently. |

---

## 3.2 Impact

Impact represents the potential effect on confidentiality, integrity, availability, operations, or security governance.

| Score | Rating | Description |
|---|---|---|
| 1 | Insignificant | Minimal operational or security consequence. |
| 2 | Minor | Limited effect with straightforward recovery. |
| 3 | Moderate | Noticeable service, data, or operational impact. |
| 4 | Major | Significant compromise, outage, data exposure, or recovery effort. |
| 5 | Severe | Critical compromise, major data loss, prolonged outage, or widespread security impact. |

---

## 3.3 Risk Calculation

Risk Score:

Likelihood × Impact

| Score | Risk Level |
|---|---|
| 1–4 | Low |
| 5–9 | Medium |
| 10–16 | High |
| 17–25 | Critical |

---

# 4. Existing Positive Security Controls

The assessment identified several security controls already implemented in the LearningSteps environment.

These controls must be considered when determining risk likelihood and impact.

## Azure and Identity Controls

- AKS uses a SystemAssigned managed identity.
- Azure Container Registry administrative user is disabled.
- AKS access uses Azure-managed identities and Azure RBAC components.
- ACR access includes an AcrPull role assignment.
- Azure Policy assignments exist at subscription level.
- Terraform state is stored remotely in Azure Storage.

## Container Security Controls

The LearningSteps application container:

- runs as a non-root user;
- uses UID 1000;
- has privilege escalation disabled;
- drops all Linux capabilities;
- has CPU requests and limits;
- has memory requests and limits;
- includes Kubernetes readiness and liveness probes.

Observed resource controls:

- CPU request: 100m
- CPU limit: 200m
- Memory request: 128Mi
- Memory limit: 256Mi

## Deployment Controls

The active application image uses a Git commit SHA:

`42fe5ccc53921a2b56863398c5774823dd7cec46`

The corresponding ACR digest was identified as:

`sha256:1fdece891fb42abca0dcb2627434e37c4ee12634ca8779c30c096144360dfaa3`

This provides traceability between source code and the deployed container artifact.

## Database Security Controls

PostgreSQL:

- requires secure transport;
- has `require_secure_transport = on`;
- requires a minimum TLS version of TLS 1.2;
- uses system-managed encryption at rest;
- has seven-day backup retention.

## Terraform State Protection

The Terraform state storage account has:

- HTTPS-only enabled;
- minimum TLS version TLS 1.2;
- blob encryption enabled;
- Microsoft-managed encryption keys;
- public blob access disabled.

## CI/CD Security Controls

The infrastructure pipeline includes:

- Terraform validation;
- Terraform format checking;
- Trivy infrastructure/configuration scanning;
- tfsec scanning;
- GitHub pull-request support;
- GitHub production environment use;
- saved Terraform plans before apply.

---

# 5. Risk Register

---

## RISK-01 — Public API Traffic Is Not Protected by HTTPS

### Asset

LearningSteps API / AKS LoadBalancer

### Observed Condition

The LearningSteps API is exposed through a Kubernetes LoadBalancer with public IP:

`20.73.209.246`

The service exposes TCP port 80.

External testing confirmed:

`HTTP /health → 200 OK`

HTTPS testing on TCP port 443 failed with a connection timeout.

Therefore, HTTPS/TLS is not currently available on the tested public application endpoint.

### Threat Scenario

An attacker positioned on a network path between a client and the LearningSteps API could potentially observe or manipulate plaintext HTTP traffic.

### Existing Controls

- PostgreSQL backend connections require TLS.
- Container executes as non-root.
- Privilege escalation is disabled.
- Linux capabilities are dropped.

These controls do not protect external HTTP traffic.

### Likelihood

4 — Likely

### Likelihood Rationale

The application is directly reachable through a public Azure LoadBalancer and HTTP is actively available.

No TLS termination layer was identified.

The exposure therefore exists whenever clients communicate with the public API.

### Impact

4 — Major

### Impact Rationale

Plaintext application traffic could affect confidentiality and integrity if sensitive information or authenticated functionality is introduced or transmitted through the API.

### Inherent Risk Score

4 × 4 = 16

### Risk Level

HIGH

### Recommended Treatment

Implement HTTPS using an ingress or gateway capable of TLS termination.

Use a trusted certificate and redirect HTTP traffic to HTTPS.

### Relevant ISO/IEC 27001:2022 Controls

- A.8.20 Network security
- A.8.21 Security of network services
- A.8.24 Use of cryptography

---

## RISK-02 — AKS Kubernetes API Is Publicly Reachable

### Asset

AKS control plane

### Observed Condition

AKS configuration showed:

- Private cluster: false
- Authorized IP ranges: `0.0.0.0/0`

The Kubernetes API therefore uses a public endpoint without source IP restriction.

### Threat Scenario

Internet-based attackers can reach the Kubernetes API endpoint and attempt authentication attacks, exploit future control-plane vulnerabilities, or abuse compromised credentials.

### Existing Controls

- Kubernetes RBAC is enabled.
- Azure authentication controls access.
- Successful authentication is still required.

### Likelihood

3 — Possible

### Likelihood Rationale

The endpoint is publicly reachable, increasing attacker opportunity.

However, public reachability does not itself provide authorization, and Kubernetes RBAC remains enabled.

### Impact

5 — Severe

### Impact Rationale

Compromise of the Kubernetes control plane could allow modification or destruction of workloads, access to secrets, deployment of malicious workloads, or service disruption.

### Inherent Risk Score

3 × 5 = 15

### Risk Level

HIGH

### Recommended Treatment

Restrict Kubernetes API authorized IP ranges or migrate to a private AKS cluster where appropriate.

Reduce privileged administrative access.

### Relevant ISO/IEC 27001:2022 Controls

- A.5.15 Access control
- A.5.18 Access rights
- A.8.20 Network security
- A.8.21 Security of network services

---

## RISK-03 — PostgreSQL Public Network Exposure

### Asset

Azure Database for PostgreSQL Flexible Server

### Observed Condition

PostgreSQL configuration showed:

`PublicNetworkAccess = Enabled`

Firewall configuration contained:

`AllowAzureServices 0.0.0.0 → 0.0.0.0`

### Threat Scenario

The database is exposed through a public Azure database endpoint and could receive connection attempts from networks permitted by its firewall configuration.

Compromised Azure-hosted workloads or credentials could potentially be used to attempt database access.

### Existing Controls

- PostgreSQL requires secure transport.
- Minimum TLS version is TLS 1.2.
- Database authentication remains required.
- Data at rest is encrypted.

### Likelihood

3 — Possible

### Likelihood Rationale

The database has public network access, but authentication and TLS significantly reduce direct exploitation opportunities.

### Impact

5 — Severe

### Impact Rationale

Successful database compromise could expose, modify, or destroy LearningSteps application data.

### Inherent Risk Score

3 × 5 = 15

### Risk Level

HIGH

### Recommended Treatment

Evaluate use of private networking/private endpoints for PostgreSQL.

Restrict firewall access to only required network paths.

Avoid broad Azure-service access where unnecessary.

### Relevant ISO/IEC 27001:2022 Controls

- A.5.15 Access control
- A.8.20 Network security
- A.8.21 Security of network services
- A.8.24 Use of cryptography

---

## RISK-04 — No Kubernetes NetworkPolicy Is Implemented

### Asset

AKS workloads

### Observed Condition

AKS uses the Azure network policy engine.

However:

`kubectl get networkpolicies`

returned:

`No resources found in default namespace.`

### Threat Scenario

If one Kubernetes workload is compromised, unrestricted pod networking may permit lateral communication to other workloads or services that would otherwise be isolated.

### Existing Controls

- Azure network policy capability is enabled.
- Current environment contains a limited number of workloads.

### Likelihood

3 — Possible

### Impact

4 — Major

### Inherent Risk Score

12

### Risk Level

HIGH

### Recommended Treatment

Implement default-deny NetworkPolicies and explicitly permit required ingress and egress paths.

### Relevant ISO/IEC 27001:2022 Controls

- A.8.20 Network security
- A.8.22 Segregation of networks

---

## RISK-05 — Application Uses Default Kubernetes Service Account and Mounted API Token

### Asset

LearningSteps application pods

### Observed Condition

Both LearningSteps pods use:

`serviceAccount=default`

The service-account token is mounted at:

`/var/run/secrets/kubernetes.io/serviceaccount`

Effective RBAC testing showed the default service account does not currently have broad Kubernetes resource permissions.

### Threat Scenario

If the application is compromised, an attacker could obtain the mounted Kubernetes service-account token and use whatever API permissions are associated with that identity.

### Existing Controls

Effective RBAC permissions are currently minimal.

The service account does not have cluster-admin privileges.

### Likelihood

2 — Unlikely

### Impact

3 — Moderate

### Inherent Risk Score

6

### Risk Level

MEDIUM

### Recommended Treatment

Create a dedicated service account for LearningSteps.

If the application does not require Kubernetes API access, configure:

`automountServiceAccountToken: false`

### Relevant ISO/IEC 27001:2022 Controls

- A.5.15 Access control
- A.5.16 Identity management
- A.5.18 Access rights
- A.8.2 Privileged access rights

---

## RISK-06 — No Kubernetes Pod Security Admission Enforcement Identified

### Asset

AKS default namespace

### Observed Condition

The default namespace contained only the standard metadata label.

No Pod Security Admission enforcement label was identified.

The current application workload nevertheless implements several strong container security controls.

### Threat Scenario

Future workloads could be deployed without the security settings currently present in the LearningSteps Deployment because the namespace does not enforce a security baseline.

### Existing Controls

Current container:

- runs as non-root;
- prevents privilege escalation;
- drops all capabilities.

### Likelihood

3 — Possible

### Impact

4 — Major

### Inherent Risk Score

12

### Risk Level

HIGH

### Recommended Treatment

Evaluate Kubernetes Pod Security Admission.

Apply an appropriate `baseline` or `restricted` security standard after compatibility testing.

### Relevant ISO/IEC 27001:2022 Controls

- A.8.9 Configuration management
- A.8.19 Installation of software on operational systems

---

## RISK-07 — Container Root Filesystem Is Writable

### Asset

LearningSteps application container

### Observed Condition

The container security context showed:

`readOnlyRootFilesystem: false`

### Threat Scenario

Following application compromise, an attacker may be able to write files within writable portions of the container filesystem.

### Existing Controls

- Non-root execution
- UID 1000
- privilege escalation disabled
- all Linux capabilities dropped

### Likelihood

2 — Unlikely

### Impact

3 — Moderate

### Inherent Risk Score

6

### Risk Level

MEDIUM

### Recommended Treatment

Test application compatibility with:

`readOnlyRootFilesystem: true`

Provide explicit writable temporary volumes only where required.

### Relevant ISO/IEC 27001:2022 Controls

- A.8.9 Configuration management
- A.8.19 Installation of software on operational systems

---

## RISK-08 — Python Dependencies Are Not Version Pinned

### Asset

Application software supply chain

### Observed Condition

`app/requirements.txt` contains package names without explicit versions, including:

- fastapi
- uvicorn
- python-dotenv
- aiohttp
- sqlalchemy
- psycopg2-binary
- asyncpg
- prometheus-client

### Threat Scenario

Application builds performed at different times may resolve different dependency versions.

Unexpected dependency updates could introduce incompatibilities or vulnerable software.

### Existing Controls

Dependencies are installed during controlled container builds.

### Likelihood

3 — Possible

### Impact

4 — Major

### Inherent Risk Score

12

### Risk Level

HIGH

### Recommended Treatment

Pin approved dependency versions.

Introduce a controlled dependency update process and automated dependency vulnerability scanning.

### Relevant ISO/IEC 27001:2022 Controls

- A.8.8 Management of technical vulnerabilities
- A.8.9 Configuration management
- A.8.25 Secure development life cycle
- A.8.29 Security testing in development and acceptance

---

## RISK-09 — Application Container Vulnerability Scanning Is Not Demonstrated in Current Pipeline

### Asset

CI/CD pipeline and application container

### Observed Condition

The current infrastructure workflow uses Trivy for configuration scanning and tfsec for Terraform scanning.

No application dependency scan or built container image vulnerability scan was identified in the active application deployment workflow.

The project README describes image scanning behavior that does not match the current workflow implementation.

### Threat Scenario

Known vulnerabilities in operating-system packages or Python dependencies could reach the production container without automated detection.

### Existing Controls

- IaC scanning exists.
- Containers use a slim Python base image.
- OS packages are updated during build.
- Container runs as non-root.

### Likelihood

3 — Possible

### Impact

4 — Major

### Inherent Risk Score

12

### Risk Level

HIGH

### Recommended Treatment

Add application dependency and built-image vulnerability scanning to CI/CD.

Fail builds according to an approved severity policy.

### Relevant ISO/IEC 27001:2022 Controls

- A.8.8 Management of technical vulnerabilities
- A.8.25 Secure development life cycle
- A.8.29 Security testing in development and acceptance

---

## RISK-10 — Mutable GitHub Action Reference Used

### Asset

GitHub Actions CI/CD pipeline

### Observed Condition

The infrastructure workflow references:

`aquasecurity/trivy-action@master`

The `master` branch is mutable.

### Threat Scenario

Code executed by the CI/CD workflow could change without a corresponding change to the LearningSteps repository.

A compromised or unexpectedly modified upstream action could affect the build pipeline.

### Existing Controls

Other GitHub Actions generally use version tags.

### Likelihood

2 — Unlikely

### Impact

4 — Major

### Inherent Risk Score

8

### Risk Level

MEDIUM

### Recommended Treatment

Pin security-sensitive GitHub Actions to reviewed immutable commit SHAs.

Use controlled dependency update tooling/processes.

### Relevant ISO/IEC 27001:2022 Controls

- A.5.21 Managing information security in the ICT supply chain
- A.8.25 Secure development life cycle
- A.8.30 Outsourced development

---

## RISK-11 — No .dockerignore File Identified

### Asset

Container build process

### Observed Condition

No `app/.dockerignore` file exists.

The application pipeline builds using the `app` directory as Docker build context.

### Threat Scenario

Unnecessary or sensitive files placed in the application directory could unintentionally become part of the Docker build context.

### Existing Controls

No current evidence demonstrated that secrets are copied into the container image.

Repository secret searches did not identify common credential files within the assessed paths except separately reviewed configuration artifacts.

### Likelihood

2 — Unlikely

### Impact

3 — Moderate

### Inherent Risk Score

6

### Risk Level

MEDIUM

### Recommended Treatment

Create an explicit `.dockerignore` containing unnecessary development files, environment files, credentials, caches, Git metadata, and other non-build artifacts.

### Relevant ISO/IEC 27001:2022 Controls

- A.8.9 Configuration management
- A.8.12 Data leakage prevention
- A.8.25 Secure development life cycle

---

## RISK-12 — Centralized Security Logging Is Not Configured for LearningSteps

### Asset

AKS and PostgreSQL

### Observed Condition

AKS diagnostic settings returned:

`[]`

PostgreSQL diagnostic settings returned:

`[]`

AKS OMS Agent and Defender add-on values were not identified as enabled.

A Log Analytics workspace exists:

`governance-lab-logs`

with 30-day retention.

However, no evidence demonstrated that LearningSteps AKS or PostgreSQL logs are being sent to that workspace.

### Threat Scenario

Security incidents may occur without sufficient durable logging for detection, investigation, or reconstruction.

Pod-local logs may disappear when workloads are recreated.

### Existing Controls

The application generates:

- Uvicorn access logs;
- startup logs;
- CRUD event logs;
- warning events.

### Likelihood

4 — Likely

### Impact

4 — Major

### Inherent Risk Score

16

### Risk Level

HIGH

### Recommended Treatment

Configure Azure Monitor diagnostic settings for AKS and PostgreSQL.

Forward security-relevant logs to a centralized Log Analytics workspace.

Define retention requirements.

Implement security alerting for high-risk events.

### Relevant ISO/IEC 27001:2022 Controls

- A.8.15 Logging
- A.8.16 Monitoring activities

---

## RISK-13 — Limited Database High Availability and Recovery Resilience

### Asset

PostgreSQL Flexible Server

### Observed Condition

PostgreSQL configuration showed:

- Backup retention: 7 days
- Geo-redundant backup: Disabled
- High availability: Disabled
- Storage auto-grow: Disabled
- Availability zone: 1

No documented RPO, RTO, restore procedure, or restore-test evidence was identified.

### Threat Scenario

A database outage, storage exhaustion, corruption event, or infrastructure failure could cause extended application unavailability or data recovery challenges.

### Existing Controls

Seven-day automated backup retention is enabled.

### Likelihood

3 — Possible

### Impact

4 — Major

### Inherent Risk Score

12

### Risk Level

HIGH

### Recommended Treatment

Define RPO and RTO requirements.

Document and test database restoration.

Evaluate PostgreSQL high availability and geo-redundant backup according to business requirements.

Evaluate storage auto-grow.

### Relevant ISO/IEC 27001:2022 Controls

- A.5.30 ICT readiness for business continuity
- A.8.13 Information backup
- A.8.14 Redundancy of information processing facilities

---

## RISK-14 — Application Replicas Share the Same AKS Node

### Asset

LearningSteps application availability

### Observed Condition

Two application replicas are configured.

However, both running pods were scheduled on:

`aks-default-27478861-vmss000002`

A second Ready node existed:

`aks-default-27478861-vmss000003`

No pod anti-affinity or topology spread constraints were configured.

### Threat Scenario

Failure of the node hosting both application replicas could make all application replicas unavailable until Kubernetes reschedules them.

### Existing Controls

- Two replicas exist.
- RollingUpdate is enabled.
- `maxUnavailable=0`.
- Two AKS nodes are available.

### Likelihood

3 — Possible

### Impact

3 — Moderate

### Inherent Risk Score

9

### Risk Level

MEDIUM

### Recommended Treatment

Configure topology spread constraints or pod anti-affinity to distribute replicas across nodes and, where applicable, availability zones.

### Relevant ISO/IEC 27001:2022 Controls

- A.5.30 ICT readiness for business continuity
- A.8.14 Redundancy of information processing facilities

---

## RISK-15 — Terraform State Storage Network Access Is Broad

### Asset

Terraform remote state

### Observed Condition

Terraform state is stored in Azure Storage.

Positive controls include:

- HTTPS-only enabled;
- TLS 1.2 minimum;
- blob public access disabled;
- blob encryption enabled.

However, the storage account network rule default action was:

`Allow`

### Threat Scenario

The storage service is network-accessible more broadly than necessary.

If valid credentials are compromised, an attacker may have an easier network path to attempt access to sensitive Terraform state.

### Existing Controls

- Authentication required.
- Anonymous blob access disabled.
- Encryption in transit enforced.
- Encryption at rest enabled.

### Likelihood

2 — Unlikely

### Impact

4 — Major

### Inherent Risk Score

8

### Risk Level

MEDIUM

### Recommended Treatment

Evaluate restricting storage network access using approved networks, private endpoints, or equivalent controls.

Maintain strong RBAC on Terraform state.

### Relevant ISO/IEC 27001:2022 Controls

- A.5.15 Access control
- A.8.20 Network security
- A.8.24 Use of cryptography

---

## RISK-16 — ACR Public Network Access Is Enabled

### Asset

Azure Container Registry

### Observed Condition

ACR configuration showed:

- Admin user: False
- Public network access: Enabled

### Threat Scenario

The registry is reachable through a public network endpoint.

Attackers with compromised credentials may therefore have a direct network path to attempt registry operations.

### Existing Controls

- ACR administrative account is disabled.
- Authentication is required.
- AcrPull least-privilege role assignment exists.

### Likelihood

2 — Unlikely

### Impact

4 — Major

### Inherent Risk Score

8

### Risk Level

MEDIUM

### Recommended Treatment

Evaluate private ACR networking where justified.

Restrict public access according to operational requirements.

### Relevant ISO/IEC 27001:2022 Controls

- A.5.15 Access control
- A.8.20 Network security
- A.8.21 Security of network services

---

## RISK-17 — Kubernetes Local Accounts Remain Enabled

### Asset

AKS administrative access

### Observed Condition

AKS reported:

`disableLocalAccounts = false`

The kubeconfig context used:

`clusterUser_rg-dev-aks_aks-west-eu`

### Threat Scenario

Local Kubernetes credentials provide an additional authentication path outside the preferred centralized identity model.

Compromise or uncontrolled distribution of local credentials could allow cluster access.

### Existing Controls

- Azure identity integration is available.
- Kubernetes RBAC is enabled.

### Likelihood

3 — Possible

### Impact

5 — Severe

### Inherent Risk Score

15

### Risk Level

HIGH

### Recommended Treatment

Evaluate disabling AKS local accounts after validating Azure identity-based administrative access.

Use centralized identity, RBAC, MFA, and controlled privileged access.

### Relevant ISO/IEC 27001:2022 Controls

- A.5.15 Access control
- A.5.16 Identity management
- A.5.17 Authentication information
- A.5.18 Access rights
- A.8.2 Privileged access rights

---

## RISK-18 — Broad Azure Subscription Privileges

### Asset

Azure subscription

### Observed Condition

Subscription-level role assignments included multiple Owner and Contributor principals.

The assessor account itself held Owner privileges.

Several service principals held Contributor permissions.

### Threat Scenario

Compromise or misuse of a highly privileged identity could permit modification, deletion, or compromise of a large portion of the Azure environment.

### Existing Controls

Azure RBAC is implemented.

### Likelihood

3 — Possible

### Impact

5 — Severe

### Inherent Risk Score

15

### Risk Level

HIGH

### Recommended Treatment

Review Owner and Contributor assignments.

Apply least privilege.

Use narrower resource scopes where possible.

Periodically review privileged identities and remove unnecessary assignments.

Consider privileged identity management where available.

### Relevant ISO/IEC 27001:2022 Controls

- A.5.15 Access control
- A.5.16 Identity management
- A.5.18 Access rights
- A.8.2 Privileged access rights

---

## RISK-19 — Kubernetes Secret Is Not Integrated with Centralized Secrets Management

### Asset

Database credential / Kubernetes secret

### Observed Condition

The application obtains its database URL from:

`learningsteps-secrets`

Secret type:

`Opaque`

Key:

`database-url`

No Azure Key Vault was identified during the subscription assessment.

The secret value was not exposed during testing.

### Threat Scenario

A Kubernetes or cluster compromise could expose application secrets stored within Kubernetes.

Secret rotation and centralized lifecycle management may also be more difficult.

### Existing Controls

- Secret is not embedded directly in the Deployment.
- Repository checks did not identify the database password committed in Terraform variables.
- PostgreSQL password was provided through an environment variable during Terraform operations.

### Likelihood

3 — Possible

### Impact

5 — Severe

### Inherent Risk Score

15

### Risk Level

HIGH

### Recommended Treatment

Evaluate Azure Key Vault and AKS workload identity/Secrets Store CSI integration.

Define credential rotation procedures.

Avoid long-lived static credentials where practical.

### Relevant ISO/IEC 27001:2022 Controls

- A.5.16 Identity management
- A.5.17 Authentication information
- A.8.2 Privileged access rights
- A.8.24 Use of cryptography

---

## RISK-20 — Security Documentation Does Not Fully Match Current Implementation

### Asset

Security documentation / CI/CD governance

### Observed Condition

`app/README.md` describes security scanning and Kubernetes authentication behavior that does not match the active GitHub Actions workflow inspected during the assessment.

For example, documentation describes application/image scanning, while the current workflow evidence did not demonstrate those controls.

### Threat Scenario

Operators, reviewers, and auditors may rely on controls documented as implemented when the production pipeline no longer performs them.

### Existing Controls

Git version history exists and the current workflows are stored as code.

### Likelihood

3 — Possible

### Impact

3 — Moderate

### Inherent Risk Score

9

### Risk Level

MEDIUM

### Recommended Treatment

Update security and deployment documentation whenever CI/CD architecture changes.

Include documentation review in the change-management process.

### Relevant ISO/IEC 27001:2022 Controls

- A.5.1 Policies for information security
- A.5.37 Documented operating procedures
- A.8.9 Configuration management

---

# 6. Risk Summary

| Risk ID | Risk | Likelihood | Impact | Score | Level |
|---|---|---:|---:|---:|---|
| RISK-01 | Public API without HTTPS | 4 | 4 | 16 | High |
| RISK-02 | Public AKS API endpoint | 3 | 5 | 15 | High |
| RISK-03 | PostgreSQL public network exposure | 3 | 5 | 15 | High |
| RISK-04 | No Kubernetes NetworkPolicy | 3 | 4 | 12 | High |
| RISK-05 | Default service account/token mount | 2 | 3 | 6 | Medium |
| RISK-06 | No Pod Security Admission enforcement | 3 | 4 | 12 | High |
| RISK-07 | Writable container root filesystem | 2 | 3 | 6 | Medium |
| RISK-08 | Unpinned Python dependencies | 3 | 4 | 12 | High |
| RISK-09 | Container vulnerability scanning not demonstrated | 3 | 4 | 12 | High |
| RISK-10 | Mutable GitHub Action reference | 2 | 4 | 8 | Medium |
| RISK-11 | No `.dockerignore` | 2 | 3 | 6 | Medium |
| RISK-12 | No centralized LearningSteps diagnostic logging | 4 | 4 | 16 | High |
| RISK-13 | Limited database HA/recovery maturity | 3 | 4 | 12 | High |
| RISK-14 | Application replicas on same node | 3 | 3 | 9 | Medium |
| RISK-15 | Broad Terraform state network access | 2 | 4 | 8 | Medium |
| RISK-16 | ACR public network access | 2 | 4 | 8 | Medium |
| RISK-17 | AKS local accounts enabled | 3 | 5 | 15 | High |
| RISK-18 | Broad Azure subscription privileges | 3 | 5 | 15 | High |
| RISK-19 | Kubernetes secret not centrally managed | 3 | 5 | 15 | High |
| RISK-20 | Documentation/implementation discrepancy | 3 | 3 | 9 | Medium |

---

# 7. Risk Prioritization

## Priority 1 — Immediate / High-Value Controls

The following should receive early remediation attention:

1. Implement HTTPS/TLS for the public LearningSteps API.
2. Establish centralized AKS and PostgreSQL logging.
3. Review and reduce privileged Azure RBAC assignments.
4. Restrict AKS API exposure.
5. Review PostgreSQL public network exposure.
6. Improve Kubernetes secrets management.
7. Implement Kubernetes network isolation.
8. Introduce application dependency and container image vulnerability scanning.

---

## Priority 2 — Platform Hardening

1. Evaluate disabling AKS local accounts.
2. Implement Pod Security Admission.
3. Introduce dedicated application service accounts.
4. Disable unnecessary service-account token mounting.
5. Evaluate read-only container filesystems.
6. Pin Python dependencies.
7. Pin GitHub Actions more strongly.
8. Add `.dockerignore`.
9. Restrict Terraform state network access.
10. Evaluate private ACR networking.

---

## Priority 3 — Resilience and Governance

1. Define RPO and RTO.
2. Document database restore procedures.
3. Test backup restoration.
4. Evaluate PostgreSQL HA and geo-redundant backup.
5. Spread application replicas across nodes.
6. Implement resource ownership/environment tags.
7. Evaluate resource locks for critical infrastructure.
8. Align README/security documentation with actual CI/CD behavior.

---

# 8. Important Assessment Limitations

This assessment did not perform penetration testing.

No attempt was made to exploit the identified weaknesses.

No password cracking, application exploitation, privilege escalation, destructive testing, denial-of-service testing, or database exploitation was performed.

Container vulnerability scanning of the deployed image was not performed locally because Trivy was not installed and the Docker daemon was unavailable during evidence collection.

Therefore, the assessment does not claim that exploitable container vulnerabilities exist.

Microsoft Sentinel status was not established because the installed Azure CLI did not recognize the attempted Sentinel command.

PostgreSQL TLS configuration was subsequently verified after the database became available.

The confirmed configuration was:

- `require_secure_transport = on`
- `ssl_min_protocol_version = TLSv1.2`

---

# 9. Overall Risk Assessment

The LearningSteps environment demonstrates several good foundational security practices, particularly container least privilege, encrypted PostgreSQL transport, encrypted storage, infrastructure-as-code validation, managed identity use, and deployment traceability.

However, significant risks remain around public network exposure, plaintext HTTP application traffic, centralized monitoring, Kubernetes policy enforcement, privileged access, secrets management, software supply-chain controls, and resilience.

Overall security maturity is therefore assessed as:

**PARTIALLY EFFECTIVE — IMPROVEMENT REQUIRED**

The environment has meaningful security controls but requires additional defense-in-depth controls before it should be considered appropriately hardened for a production workload handling sensitive or business-critical information.

---

# 10. Next Phase

The next phase of the LearningSteps GRC project will:

1. Validate ISO/IEC 27001:2022 control mappings.
2. Perform formal gap analysis.
3. Prioritize security improvements.
4. Define control implementation activities.
5. Assign control ownership.
6. Establish target completion dates.
7. Implement selected security controls.
8. Reassess residual risk after implementation.

A separate penetration-testing project may later validate exploitable attack paths after the security control improvement phase.
