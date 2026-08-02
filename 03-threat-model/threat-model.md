# LearningSteps GRC Assessment — Evidence-Informed Threat Model

## 1. Purpose

This threat model identifies realistic security threats that could affect
the LearningSteps environment.

The model is informed by the technical security assessment documented in:

    evidence/security-assessment.md

The objective is to establish traceability between:

    Technical Evidence
          ↓
    Security Finding
          ↓
    Threat Scenario
          ↓
    Risk Assessment
          ↓
    ISO/IEC 27001 Control
          ↓
    Security Recommendation

The threat model does not assign likelihood or risk ratings.

Likelihood and impact will be evaluated separately in the formal risk
assessment.

This distinction prevents assumptions from being presented as measured
risk.

---

## 2. Scope

The threat model covers the LearningSteps production architecture,
including:

- FastAPI application;
- Azure Load Balancer;
- Azure Kubernetes Service (AKS);
- Kubernetes workloads;
- Kubernetes RBAC;
- Kubernetes Secrets;
- Azure Database for PostgreSQL Flexible Server;
- Azure Container Registry;
- GitHub source repository;
- GitHub Actions;
- Terraform infrastructure;
- Terraform remote state;
- Azure identities and service principals; and
- CI/CD deployment credentials.

The threat model is based on security configuration and control evidence.

It is not based on penetration testing.

---

## 3. Threat-Modelling Method

Threats are divided into three evidence categories.

### 3.1 Evidence-Supported Threat

A technical condition relevant to the threat was directly observed
during the security assessment.

Example:

    Evidence:
    AKS authorized IP ranges include 0.0.0.0/0.

    Threat:
    An external actor can reach the Kubernetes API endpoint and attempt
    attacks against the cluster authentication boundary.

This does not mean successful exploitation was demonstrated.

---

### 3.2 Architecture-Based Threat

The system architecture makes the threat reasonably applicable, but the
technical assessment did not attempt to exploit or validate the threat.

Example:

    Public web application
            ↓
    Web application vulnerability exploitation

The threat remains relevant even though no penetration testing was
performed.

---

### 3.3 Unverified Security Concern

The security area is relevant to the system but was not sufficiently
tested during SA-01 through SA-04.

These concerns must not be represented as confirmed vulnerabilities.

Examples include:

- backup effectiveness;
- centralized security monitoring;
- application-level authorization weaknesses;
- exploitable software vulnerabilities; and
- incident-response effectiveness.

---

## 4. Threat Sources

### External Attacker

An individual or group outside the organization attempting to exploit
publicly reachable infrastructure or application services.

Potential objectives include:

- unauthorized access;
- information disclosure;
- service disruption;
- credential theft; and
- infrastructure compromise.

---

### Compromised Credential

A legitimate Azure, GitHub, Kubernetes, database or CI/CD credential
obtained by an unauthorized party.

The attacker may be able to bypass portions of the external security
perimeter by authenticating as a legitimate identity.

---

### Malicious or Unauthorized User

A legitimate or partially authorized user who attempts to access,
modify or delete information outside their intended authorization.

---

### Malicious Insider

An individual with legitimate administrative, development or deployment
access who intentionally misuses that access.

---

### Human Error or Misconfiguration

Security exposure introduced unintentionally through:

- Azure configuration;
- Kubernetes configuration;
- firewall configuration;
- identity configuration;
- Terraform;
- GitHub Actions;
- secrets handling; or
- application deployment.

---

### Software Supply-Chain Threat Actor

An attacker who compromises a software dependency, build component,
container image source or CI/CD dependency used by LearningSteps.

---

## 5. Primary Assets at Risk

The primary assets considered by this threat model include:

### Application

The LearningSteps FastAPI application and its functionality.

### Learning Data

Information stored and processed by the application.

### PostgreSQL Database

The persistent database supporting LearningSteps.

### AKS Cluster

The Kubernetes control plane, worker nodes, workloads and configuration.

### Kubernetes Secrets

Credentials and connection information stored in Kubernetes.

### Azure Subscription Resources

Cloud infrastructure supporting LearningSteps.

### GitHub Repository

Application source code, infrastructure code and CI/CD workflow
definitions.

### CI/CD Pipeline

GitHub Actions workflows capable of building and deploying application
and infrastructure changes.

### Azure Container Registry

Container images used by the LearningSteps application.

### Terraform State

Infrastructure state and metadata stored in Azure Storage.

### Administrative Credentials

Azure, GitHub, Kubernetes, database and CI/CD credentials capable of
accessing or modifying LearningSteps resources.

---

# 6. Evidence-Supported Threats

## T-01 — Unauthorized Attempts Against the Public Application

### Threat Source

External attacker

### Target Assets

- LearningSteps API
- application data
- application availability

### Supporting Evidence

The technical assessment confirmed that:

- `learningsteps-api` is exposed using a Kubernetes `LoadBalancer`;
- an Azure public IP address is assigned;
- the service is reachable from the Internet;
- the service listens publicly on HTTP port 80; and
- no Service-level `loadBalancerSourceRanges` restriction was identified.

### Threat Scenario

An external attacker discovers the public LearningSteps endpoint and
interacts with application functionality from the Internet.

If an application-level vulnerability or authorization weakness exists,
the public network path provides the attacker with an opportunity to
attempt exploitation.

### Potential Consequences

- unauthorized application access;
- information disclosure;
- unauthorized modification;
- application abuse; or
- service disruption.

### Evidence Status

**Exposure confirmed. Exploitation not tested.**

---

## T-02 — Interception or Manipulation of Unencrypted Application Traffic

### Threat Source

Network-positioned attacker / misconfiguration

### Target Assets

- application communications;
- user information;
- session or application data.

### Supporting Evidence

The assessment confirmed:

- HTTP access on port 80 was functional; and
- HTTPS access on port 443 was not available during testing.

### Threat Scenario

Application traffic transmitted without TLS may be exposed to
interception or manipulation by an attacker capable of observing or
altering the network path.

### Potential Consequences

- loss of confidentiality;
- loss of integrity;
- exposure of application information; or
- manipulation of communications.

### Evidence Status

**Lack of HTTPS on the assessed public endpoint confirmed.**

---

## T-03 — Attack Against the Public AKS API Endpoint

### Threat Source

External attacker

### Target Assets

- AKS control plane;
- Kubernetes workloads;
- Kubernetes Secrets;
- application availability.

### Supporting Evidence

The technical assessment confirmed:

    PrivateCluster: false

and:

    AuthorizedIPRanges:
      - 0.0.0.0/0

### Threat Scenario

Because the Kubernetes management endpoint has a broad network exposure,
an external actor can reach the API endpoint and attempt attacks against
the authentication boundary.

Network reachability alone does not grant access.

Successful compromise would still require valid credentials, an
authentication weakness or another exploitable condition.

### Potential Consequences

If successful:

- cluster compromise;
- workload modification;
- Secret access;
- malicious workload deployment;
- application disruption; or
- lateral movement.

### Evidence Status

**Network exposure confirmed. Unauthorized access not demonstrated.**

---

## T-04 — Misuse or Compromise of Highly Privileged Azure Access

### Threat Source

Compromised credential / malicious insider

### Target Assets

- Azure subscription;
- AKS;
- PostgreSQL;
- networking;
- storage;
- other LearningSteps resources.

### Supporting Evidence

The technical assessment identified:

- multiple human identities with Owner access;
- multiple service principals with Contributor access;
- privileged access inherited from higher Azure scopes.

### Threat Scenario

An attacker who compromises a highly privileged Azure identity, or an
authorized individual who misuses that identity, may be able to modify
or destroy LearningSteps infrastructure.

### Potential Consequences

- infrastructure modification;
- security-control modification;
- resource deletion;
- credential access;
- service disruption; or
- creation of persistent unauthorized access.

### Evidence Status

**Privileged access confirmed. Credential compromise not demonstrated.**

---

## T-05 — Kubernetes Administrative Credential Compromise

### Threat Source

Compromised credential / malicious insider

### Target Assets

- AKS cluster;
- application workloads;
- Kubernetes Secrets;
- service availability.

### Supporting Evidence

The assessment confirmed:

    kubectl auth can-i '*' '*'
    yes

Cluster-level RBAC inspection identified a binding involving:

    cluster-admin

and AKS administrative/user subjects.

Local AKS accounts were also confirmed to remain enabled.

### Threat Scenario

If a credential with the observed Kubernetes privileges is compromised,
the attacker could obtain extensive control over the Kubernetes
environment.

### Potential Consequences

- Secret retrieval;
- workload modification;
- malicious container deployment;
- deletion of resources;
- persistence;
- service disruption; or
- cluster-wide compromise.

### Evidence Status

**Administrative privilege confirmed. Credential compromise not
demonstrated.**

---

## T-06 — Unauthorized Database Network Access

### Threat Source

External attacker / compromised Azure resource / misconfiguration

### Target Assets

- PostgreSQL database;
- LearningSteps data.

### Supporting Evidence

The assessment confirmed:

    PublicNetworkAccess: Enabled

The firewall configuration included:

    AllowAzureServices
    0.0.0.0 -> 0.0.0.0

No private delegated subnet was identified through the assessed
configuration.

### Threat Scenario

The database relies on public networking and firewall controls rather
than a private application-specific network path.

An attacker would still require an allowed network path and valid
database authentication or an exploitable weakness.

### Potential Consequences

- unauthorized database connectivity;
- data disclosure;
- data modification;
- data deletion; or
- application disruption.

### Evidence Status

**Public database networking confirmed. Unauthorized database access not
demonstrated.**

---

## T-07 — Compromise of CI/CD Azure Credentials

### Threat Source

Compromised GitHub account / credential theft / malicious insider

### Target Assets

- GitHub Actions;
- Azure resources;
- AKS;
- application deployments;
- Terraform-managed infrastructure.

### Supporting Evidence

The application workflow uses:

    secrets.AZURE_CREDENTIALS

The infrastructure workflow uses:

    secrets.ARM_CLIENT_ID
    secrets.ARM_CLIENT_SECRET
    secrets.ARM_SUBSCRIPTION_ID
    secrets.ARM_TENANT_ID

Although the application workflow grants:

    id-token: write

the observed Azure login uses:

    creds: ${{ secrets.AZURE_CREDENTIALS }}

OIDC-based Azure authentication was not demonstrated.

### Threat Scenario

If a stored Azure CI/CD credential is disclosed or stolen, an attacker
may be able to authenticate to Azure with the permissions assigned to
that credential.

### Potential Consequences

- unauthorized deployments;
- infrastructure modification;
- malicious image deployment;
- security-control modification;
- resource destruction; or
- persistence.

### Evidence Status

**Stored credential use confirmed. Credential compromise not
demonstrated.**

---

## T-08 — Unauthorized CI/CD Pipeline Modification

### Threat Source

Compromised GitHub account / malicious insider

### Target Assets

- application source;
- GitHub Actions;
- container images;
- AKS;
- Azure infrastructure.

### Supporting Evidence

GitHub Actions workflows were confirmed to:

- build the LearningSteps container;
- push images to Azure Container Registry;
- change the AKS Deployment image;
- execute Terraform plan; and
- execute Terraform apply under configured conditions.

### Threat Scenario

An attacker capable of modifying trusted source code or workflow files
could attempt to introduce malicious deployment instructions into the
trusted CI/CD process.

### Potential Consequences

- malicious application deployment;
- infrastructure modification;
- credential theft;
- persistence; or
- service disruption.

### Evidence Status

**Pipeline capability confirmed. Unauthorized modification not
demonstrated.**

---

## T-09 — Kubernetes Workload Credential Exposure

### Threat Source

Compromised workload / application vulnerability

### Target Assets

- Kubernetes service-account credential;
- Kubernetes API;
- cluster resources accessible to that identity.

### Supporting Evidence

The LearningSteps pods use:

    serviceAccountName: default

The assessment also confirmed that projected service-account token
volumes are present in the application pods.

### Threat Scenario

If the application container is compromised, an attacker may be able to
access the projected Kubernetes service-account credential from the pod.

The usefulness of that credential depends on the permissions assigned
to the service account.

### Potential Consequences

Depending on effective RBAC:

- Kubernetes API access;
- resource discovery;
- unauthorized Kubernetes actions; or
- lateral movement.

### Evidence Status

**Token projection confirmed. Exploitation and effective token
permissions were not tested.**

---

## T-10 — Lateral Movement Between Kubernetes Workloads

### Threat Source

Compromised workload / malicious workload

### Target Assets

- LearningSteps pods;
- Kubernetes services;
- other cluster workloads.

### Supporting Evidence

AKS has:

    networkPolicy: azure

However:

    kubectl get networkpolicies

returned:

    No resources found in default namespace.

### Threat Scenario

If a workload in the namespace becomes compromised, the absence of
deployed NetworkPolicy controls may permit network communication that
could otherwise have been restricted.

### Potential Consequences

- lateral movement;
- unauthorized service communication;
- reconnaissance;
- access to internal workloads; or
- expansion of a container compromise.

### Evidence Status

**Absence of NetworkPolicy in the assessed namespace confirmed.
Lateral movement not tested.**

---

## T-11 — Container Filesystem Modification Following Compromise

### Threat Source

Application exploit / malicious code

### Target Assets

- LearningSteps application container.

### Supporting Evidence

The container security context showed:

    runAsNonRoot: true
    runAsUser: 1000
    allowPrivilegeEscalation: false
    capabilities.drop: ALL
    readOnlyRootFilesystem: false

### Threat Scenario

If code execution is obtained inside the application container, the
writable root filesystem may provide an attacker with more opportunities
to modify files within the container than a read-only root filesystem
would permit.

### Existing Mitigations

The potential impact is reduced by:

- non-root execution;
- disabled privilege escalation; and
- all Linux capabilities being dropped.

### Evidence Status

**Writable root filesystem confirmed. Container compromise not
demonstrated.**

---

## T-12 — Terraform State Access

### Threat Source

Compromised Azure identity / malicious insider

### Target Assets

- Terraform state;
- infrastructure metadata;
- potentially sensitive state information.

### Supporting Evidence

The assessment confirmed:

- Terraform state is not tracked by Git;
- Azure Storage is used as the remote backend;
- HTTPS is required;
- TLS 1.2 is the minimum;
- anonymous blob public access is disabled; and
- the storage network default action is `Allow`.

### Threat Scenario

An identity with sufficient Azure Storage permissions could potentially
access Terraform state.

The broad storage network boundary increases network reachability,
although authentication and authorization remain required.

### Potential Consequences

- infrastructure information disclosure;
- exposure of state-held sensitive information;
- assistance with further attacks; or
- unauthorized state modification where permissions allow.

### Evidence Status

**State-storage configuration confirmed. Unauthorized access not
demonstrated.**

---

# 7. Architecture-Based Threats Not Exploitation-Tested

The following threats remain relevant because of the LearningSteps
architecture, but SA-01 through SA-04 did not test their exploitability.

---

## T-13 — Web Application Vulnerability Exploitation

### Threat Source

External attacker

### Target Asset

FastAPI application

### Threat Scenario

A vulnerability in application code, authentication, authorization,
input validation or another application component could potentially be
exploited through the public API.

### Potential Consequences

- unauthorized access;
- data disclosure;
- data modification;
- application compromise; or
- service disruption.

### Evidence Status

**Architecture-based threat. Application penetration testing was not
performed.**

---

## T-14 — Vulnerable or Malicious Software Dependency

### Threat Source

Software supply-chain attacker / vulnerable upstream dependency

### Target Assets

- application dependencies;
- container image;
- application runtime.

### Threat Scenario

A vulnerable or malicious dependency could enter the application build
and subsequently be deployed through the trusted CI/CD process.

### Supporting Context

The infrastructure pipeline was observed using Trivy and tfsec for
infrastructure-related security scanning.

This assessment did not establish comprehensive application dependency
security or prove that deployed application dependencies are free of
known vulnerabilities.

### Evidence Status

**Architecture-based threat. Exploitable dependency vulnerabilities were
not established.**

---

## T-15 — Malicious CI/CD Dependency or Action

### Threat Source

Software supply-chain attacker

### Target Assets

- GitHub Actions;
- build environment;
- deployment credentials.

### Threat Scenario

A compromised third-party GitHub Action or build dependency could
execute within the CI/CD environment and potentially interact with
available credentials or deployment processes.

### Supporting Context

The workflows use third-party actions including Azure, Terraform and
security-scanning actions.

One observed security scanner action references a mutable branch:

    aquasecurity/trivy-action@master

### Evidence Status

**Architecture/configuration-based threat. Action compromise was not
demonstrated.**

---

## T-16 — Denial of Service Against the Public Application

### Threat Source

External attacker

### Target Assets

- FastAPI application;
- Load Balancer;
- AKS workloads;
- application availability.

### Threat Scenario

An attacker could send excessive or expensive requests to the public
application endpoint and attempt to exhaust available resources.

### Potential Consequences

- degraded performance;
- resource exhaustion; or
- application unavailability.

### Evidence Status

**Architecture-based threat. Denial-of-service testing was not
performed.**

---

## T-17 — Unauthorized Modification or Deletion of Learning Data

### Threat Source

External attacker / unauthorized user / compromised credential

### Target Asset

LearningSteps data

### Threat Scenario

If an attacker obtains application-level or database-level write access,
LearningSteps information could be altered or deleted.

### Potential Consequences

- loss of data integrity;
- loss of availability;
- incorrect learning records; or
- recovery requirements.

### Evidence Status

**Architecture-based threat. Unauthorized data modification was not
attempted.**

---

# 8. Unverified Security Concerns

These areas are relevant but were not sufficiently assessed during the
technical security assessment.

They should therefore remain clearly distinguished from confirmed
technical findings.

---

## U-01 — Backup and Recovery Effectiveness

The assessment did not verify:

- PostgreSQL backup configuration;
- retention periods;
- restore testing;
- recovery time objectives; or
- recovery point objectives.

### Status

**Not assessed.**

---

## U-02 — Centralized Security Logging and Monitoring

The assessment did not establish the effectiveness of:

- centralized log collection;
- security alerting;
- Azure Monitor;
- Microsoft Defender for Cloud;
- SIEM integration;
- incident detection; or
- alert response.

### Status

**Not assessed.**

---

## U-03 — Application Authentication and Authorization

The technical assessment did not test:

- user authentication;
- session management;
- authorization boundaries;
- object-level authorization; or
- privilege escalation within the application.

### Status

**Not assessed.**

---

## U-04 — Application Input Validation

The assessment did not perform:

- SQL injection testing;
- command injection testing;
- malformed-input testing;
- API fuzzing; or
- other application penetration tests.

### Status

**Not assessed.**

---

## U-05 — Incident Response Effectiveness

The assessment did not test the organization's ability to:

- detect an incident;
- triage alerts;
- contain compromise;
- preserve forensic evidence;
- eradicate malicious access; or
- recover services.

### Status

**Not assessed.**

---

# 9. Existing Controls Demonstrated by Evidence

The technical assessment confirmed several controls that reduce the
threats identified above.

## Identity and Access Controls

- Azure RBAC is implemented.
- Kubernetes RBAC is enabled.
- Azure and Kubernetes authentication controls exist.

## Container Security Controls

The LearningSteps container:

- runs as a non-root user;
- uses UID 1000;
- disables privilege escalation; and
- drops all Linux capabilities.

## Secrets Controls

- database connection information is stored in a Kubernetes Secret;
- `DATABASE_URL` references the Kubernetes Secret;
- no database password was identified in tracked `terraform.tfvars`;
- Terraform uses a sensitive password variable;
- Terraform state is excluded from Git;
- CI/CD sensitive values are referenced using GitHub Actions Secrets.

## Terraform State Controls

- remote Azure Storage backend;
- HTTPS required;
- minimum TLS version 1.2;
- anonymous blob public access disabled.

## Deployment Controls

- application images are deployed using a source commit-SHA tag;
- Terraform is managed through Infrastructure as Code;
- GitHub Actions automates application and infrastructure deployment.

## Network Capability

- Azure Network Policy capability is enabled in AKS.

These controls will be considered when assessing likelihood and impact.
Their existence does not automatically mean that all related risks are
adequately treated.

---

# 10. Threat-to-Finding Traceability

| Threat | Related Technical Finding |
|---|---|
| T-01 Public application attack | F-04 Public application exposure |
| T-02 Unencrypted application traffic | F-04 Public application exposure without identified TLS |
| T-03 Public AKS API attack | F-03 Public AKS management endpoint |
| T-04 Privileged Azure identity compromise | F-01 Broad Azure privileged access |
| T-05 Kubernetes administrative credential compromise | F-02 Broad Kubernetes administrative access |
| T-06 Unauthorized database network access | F-05 PostgreSQL public networking |
| T-07 CI/CD credential compromise | F-06 Long-lived CI/CD Azure credentials |
| T-08 CI/CD pipeline modification | F-06 and observed deployment capabilities |
| T-09 Workload credential exposure | F-08 Default Kubernetes workload identity |
| T-10 Kubernetes lateral movement | F-09 No workload NetworkPolicy |
| T-11 Container filesystem modification | F-10 Writable container root filesystem |
| T-12 Terraform state access | F-07 Terraform state network boundary |
| T-13 Web application vulnerability | Architecture-based; penetration testing not performed |
| T-14 Dependency compromise | Architecture-based; vulnerability exploitation not tested |
| T-15 CI/CD supply-chain compromise | Workflow configuration / third-party actions |
| T-16 Denial of service | Architecture-based public service threat |
| T-17 Data modification or deletion | Architecture-based data integrity threat |

---

# 11. Threat Model Conclusion

The evidence-informed threat model identifies several meaningful attack
paths in the LearningSteps environment.

The strongest evidence-supported threat areas are:

- public application exposure without identified HTTPS;
- broad AKS management-plane network exposure;
- highly privileged Azure identities;
- broad Kubernetes administrative permissions;
- public PostgreSQL networking;
- long-lived CI/CD Azure credentials;
- default Kubernetes workload identity and projected tokens;
- absence of workload NetworkPolicy; and
- opportunities to strengthen Terraform state network restrictions.

At the same time, the technical assessment identified important existing
controls, particularly:

- Kubernetes container hardening;
- separation of secrets from source code;
- remote Terraform state;
- TLS protection for Terraform state storage;
- GitHub Actions secret references; and
- source-revision-based container deployment.

No successful exploitation was attempted or demonstrated.

The next phase will convert these threat scenarios into formal risks.

For each risk, the assessment will determine:

1. affected asset;
2. threat scenario;
3. existing controls;
4. evidence supporting the assessment;
5. likelihood;
6. impact;
7. overall risk rating; and
8. required treatment.

This approach ensures that risk ratings are derived from technical
evidence and documented reasoning rather than unsupported assumptions.
