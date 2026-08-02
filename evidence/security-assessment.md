# LearningSteps Technical Security Assessment

## 1. Purpose

This document records the technical security assessment performed against
the LearningSteps Azure and Kubernetes environment.

The purpose of the assessment is to collect technical evidence that can
support the subsequent Governance, Risk and Compliance (GRC) activities,
including:

- threat modelling;
- likelihood determination;
- impact assessment;
- risk scoring;
- ISO/IEC 27001 control mapping;
- gap analysis;
- security recommendations; and
- policy development.

This assessment is a security configuration and control assessment.

It is not a penetration test.

A separate penetration-testing project may be performed later.

---

## 2. Assessment Method

The assessment followed an evidence-based approach.

For each security area, the following process was used:

1. Define the security objective.
2. Execute read-only or minimally invasive assessment commands.
3. Preserve the observed results.
4. Determine what the evidence demonstrates.
5. Separate confirmed facts from assumptions.
6. Identify positive controls and security gaps.
7. Assign a preliminary control-effectiveness status.

No risk likelihood was assigned solely on the basis of assumptions.

The technical evidence collected in this assessment will be used later
to determine risk likelihood and impact.

Sensitive values such as passwords, tokens, tenant identifiers and
credential values are intentionally excluded from this document.

---

# SA-01 — Identity and Access Management

## Security Objective

Determine how administrative access to the LearningSteps Azure and
Kubernetes environments is controlled and whether the observed access
model demonstrates least privilege.

---

## SA-01.1 — Azure Assessment Identity

### Command

    az account show \
      --query "{User:user.name,Subscription:name,Tenant:tenantId}" \
      -o table

### Why This Command Was Used

The command identifies the Azure account, subscription and tenant being
used to conduct the assessment.

### Evidence

The command confirmed that the assessor was authenticated to the expected
LearningSteps Azure subscription.

The assessment account was subsequently confirmed to have the Azure
Owner role at subscription scope.

Personal identifiers and tenant identifiers are excluded from this
public evidence document.

### Observation

The assessment was conducted using an authorized privileged Azure
identity.

---

## SA-01.2 — Resource Group RBAC

### Command

    az role assignment list \
      --resource-group rg-dev-aks \
      -o table

### Why This Command Was Used

The command was used to identify Azure RBAC assignments made directly
against the LearningSteps resource group.

### Evidence

No direct role assignments were returned.

### Observation

No Azure RBAC assignments were identified directly at the
`rg-dev-aks` resource-group scope.

This did not prove that the resources had no inherited permissions,
because Azure RBAC permissions can be inherited from subscriptions and
management groups.

---

## SA-01.3 — Subscription and Inherited Azure RBAC

### Commands

    SUB_ID=$(az account show --query id -o tsv)

    az role assignment list \
      --scope "/subscriptions/$SUB_ID" \
      --include-inherited \
      --query "[].{Principal:principalName,Role:roleDefinitionName,Type:principalType,Scope:scope}" \
      -o table

### Why These Commands Were Used

Azure RBAC is hierarchical.

The test was performed to determine whether LearningSteps inherited
privileged access from subscription or management-group scopes.

### Evidence

The assessment identified:

- two human identities with Owner access at subscription scope;
- multiple service principals with Contributor access;
- User Access Administrator access inherited from a higher scope;
- additional role assignments originating from management groups.

Identifiers that are unnecessary for the public assessment have been
omitted.

### Observation

The LearningSteps environment inherits privileged Azure permissions from
higher scopes.

Compromise or misuse of an identity with Owner, Contributor or other
high-level privileges could therefore affect LearningSteps resources.

---

## SA-01.4 — Initial Kubernetes Connectivity Failure

### Initial Command

    kubectl get nodes

### Initial Evidence

The command initially failed because the Kubernetes API hostname could
not be resolved.

The error included:

    Unable to connect to the server

### Investigation Command

    az aks show \
      --resource-group rg-dev-aks \
      --name aks-west-eu \
      --query "{PowerState:powerState.code,ProvisioningState:provisioningState,FQDN:fqdn}" \
      -o table

### Evidence

The AKS cluster was:

    PowerState: Stopped
    ProvisioningState: Succeeded

### Action

The authorized assessor started the cluster:

    az aks start \
      --resource-group rg-dev-aks \
      --name aks-west-eu

The cluster state was checked again and returned:

    PowerState: Running
    ProvisioningState: Succeeded

### Retest

    kubectl get nodes

### Evidence

Two AKS worker nodes returned a `Ready` status and Kubernetes version
1.35.6.

### Observation

The original Kubernetes connection failure was caused by the AKS cluster
being intentionally stopped.

It was not treated as an IAM or Kubernetes security-control failure.

This result was retained as evidence because it demonstrates the
investigation and validation process followed during the assessment.

---

## SA-01.5 — Namespace RBAC

### Commands

    kubectl get serviceaccounts

    kubectl get roles

    kubectl get rolebindings

### Why These Commands Were Used

The commands identify Kubernetes service accounts and namespace-scoped
RBAC configuration.

### Evidence

In the `default` namespace:

- the standard `default` service account existed;
- no Role resources were identified;
- no RoleBinding resources were identified.

### Observation

No custom namespace-level RBAC configuration was identified in the
default namespace.

Because this did not explain the assessor's effective Kubernetes
permissions, cluster-level RBAC was investigated.

---

## SA-01.6 — Effective Kubernetes Permissions

### Command

    kubectl auth can-i '*' '*'

### Why This Command Was Used

The command asks the Kubernetes authorization system whether the current
credential can perform all actions against all resource types.

### Evidence

The command returned:

    yes

### Observation

The current Kubernetes credential had unrestricted Kubernetes
authorization.

Further investigation was required to determine the source of the
privilege.

---

## SA-01.7 — Kubernetes Context and Credential

### Commands

    kubectl config current-context

    kubectl config view --minify \
      -o jsonpath='{.users[0].name}{"\n"}'

### Evidence

The current context was:

    aks-west-eu

The kubeconfig credential entry was:

    clusterUser_rg-dev-aks_aks-west-eu

### Observation

The unrestricted authorization test was confirmed to have been executed
against the intended LearningSteps AKS cluster.

---

## SA-01.8 — AKS Authentication Configuration

### Commands

    az aks show \
      --resource-group rg-dev-aks \
      --name aks-west-eu \
      --query "aadProfile" \
      -o json

    az aks show \
      --resource-group rg-dev-aks \
      --name aks-west-eu \
      --query "disableLocalAccounts" \
      -o tsv

### Evidence

No `aadProfile` configuration was returned by the first query.

The local-account query returned:

    false

### Observation

Local AKS accounts are enabled.

No Microsoft Entra/Azure RBAC configuration was identified through the
`aadProfile` query during this assessment.

---

## SA-01.9 — Cluster-Level Kubernetes RBAC

### Command

    kubectl get clusterrolebindings \
      -o custom-columns='NAME:.metadata.name,ROLE:.roleRef.name,SUBJECTS:.subjects[*].name'

### Why This Command Was Used

Namespace Roles and RoleBindings did not explain why the current
credential possessed unrestricted permissions.

ClusterRoleBindings were therefore inspected.

### Evidence

The assessment identified an AKS-managed binding associating:

    aks-cluster-admin-binding
        -> cluster-admin
        -> clusterAdmin, clusterUser

### Observation

Cluster-level RBAC explains the unrestricted result obtained from:

    kubectl auth can-i '*' '*'

The assessed AKS cluster user therefore had broad Kubernetes
administrative authority.

---

## SA-01 Control Assessment

### Positive Controls

- Azure RBAC is implemented.
- Kubernetes RBAC is enabled.
- Administrative access is subject to Azure/Kubernetes authorization.
- No custom namespace Role or RoleBinding was identified in `default`.

### Security Concerns

- Multiple highly privileged Azure identities exist.
- Multiple service principals have Contributor access.
- Privileged permissions are inherited from higher Azure scopes.
- Local AKS accounts remain enabled.
- The assessed AKS cluster user has unrestricted Kubernetes permissions.
- Cluster-level RBAC associates AKS administrative/user subjects with
  `cluster-admin`.

### Preliminary Control Status

**Partially Effective**

Access-control mechanisms exist, but the observed configuration does not
fully demonstrate least privilege.

---

# SA-02 — Network Security

## Security Objective

Determine whether network access to the LearningSteps application,
Kubernetes control plane and PostgreSQL database is appropriately
restricted and whether public communications are protected.

---

## SA-02.1 — Application Service Exposure

### Command

    kubectl get service learningsteps-api -o yaml

### Why This Command Was Used

The command was used to identify how the LearningSteps API is exposed
from Kubernetes and whether source restrictions are configured.

### Evidence

The Service configuration showed:

    type: LoadBalancer
    port: 80
    targetPort: 8000
    protocol: TCP

An Azure public IP address was assigned.

A NodePort was also allocated.

No `loadBalancerSourceRanges` configuration was identified.

### Observation

The LearningSteps API has an Internet-facing network path through an
Azure LoadBalancer.

No Kubernetes Service-level source IP allowlist was identified.

---

## SA-02.2 — AKS API Server Exposure

### Command

    az aks show \
      --resource-group rg-dev-aks \
      --name aks-west-eu \
      --query "{PrivateCluster:apiServerAccessProfile.enablePrivateCluster,AuthorizedIPRanges:apiServerAccessProfile.authorizedIpRanges,PublicFQDN:apiServerAccessProfile.enablePrivateClusterPublicFQDN}" \
      -o json

### Evidence

The configuration returned:

    PrivateCluster: false
    AuthorizedIPRanges:
      - 0.0.0.0/0

### Observation

The AKS cluster is not configured as a private cluster.

The API-server source range is extremely broad rather than being limited
to known administrative networks.

Authentication and authorization are still required, but the Kubernetes
management endpoint has a broad network exposure.

---

## SA-02.3 — PostgreSQL Network Configuration

### Command

    az postgres flexible-server show \
      --resource-group rg-dev-aks \
      --name psql-dev-aks-west-eu \
      --query "{PublicNetworkAccess:network.publicNetworkAccess,DelegatedSubnet:network.delegatedSubnetResourceId,PrivateDNSZone:network.privateDnsZoneArmResourceId,FQDN:fullyQualifiedDomainName}" \
      -o table

### Evidence

The command returned:

    PublicNetworkAccess: Enabled

No delegated subnet or private DNS zone was returned.

### Observation

The PostgreSQL Flexible Server uses public network access rather than
the private VNet-integrated configuration represented by the queried
fields.

Firewall configuration was therefore assessed separately.

---

## SA-02.4 — PostgreSQL Firewall Assessment

### Initial Command

    az postgres flexible-server firewall-rule list \
      --resource-group rg-dev-aks \
      --name psql-dev-aks-west-eu \
      --query "[].{Rule:name,StartIP:startIpAddress,EndIP:endIpAddress}" \
      -o table

### Initial Result

The installed Azure CLI rejected:

    --name

This was identified as a command-syntax issue and was not treated as a
security finding.

### CLI Verification

    az postgres flexible-server firewall-rule list -h

The Azure CLI help showed that the correct argument was:

    --server-name

### Corrected Command

    az postgres flexible-server firewall-rule list \
      --resource-group rg-dev-aks \
      --server-name psql-dev-aks-west-eu \
      --query "[].{Rule:name,StartIP:startIpAddress,EndIP:endIpAddress}" \
      -o table

### Evidence

The following rule was identified:

    Rule: AllowAzureServices
    StartIP: 0.0.0.0
    EndIP: 0.0.0.0

### Observation

No arbitrary Internet IP range was identified in the collected firewall
rules.

The Azure-services firewall model nevertheless represents a broader
trust boundary than access restricted specifically to the LearningSteps
application workload.

---

## SA-02.5 — Public Transport Encryption

### HTTP Test

    curl http://<PUBLIC-IP>/health

### Evidence

The application returned:

    {"status":"healthy"}

### HTTPS Test

    curl -I --connect-timeout 10 https://<PUBLIC-IP>/health

### Evidence

The HTTPS connection to TCP port 443 timed out.

### Observation

The assessed application endpoint is reachable using HTTP on port 80.

No working HTTPS endpoint was identified on TCP port 443 during the
assessment.

The Kubernetes Service configuration also exposed port 80 rather than
443.

Transport encryption was therefore not identified for the assessed
public application endpoint.

The public IP address has been replaced with `<PUBLIC-IP>` in this
document.

---

## SA-02.6 — AKS Network Profile

### Command

    az aks show \
      --resource-group rg-dev-aks \
      --name aks-west-eu \
      --query "networkProfile" \
      -o json

### Evidence

Relevant configuration included:

    networkPlugin: azure
    networkDataplane: azure
    networkPolicy: azure
    loadBalancerSku: standard
    outboundType: loadBalancer
    ipFamilies: IPv4

A managed outbound public IP was also configured.

### Observation

Azure Network Policy capability is enabled at cluster level.

This is a positive security capability because Kubernetes NetworkPolicy
objects can be used to restrict workload communication.

The presence of the capability does not prove that NetworkPolicy
resources are deployed.

The workload assessment later confirmed that no NetworkPolicy resources
were present in the `default` namespace.

---

## SA-02 Control Assessment

### Positive Controls

- Azure Standard Load Balancer is used.
- PostgreSQL firewall functionality is enabled.
- No arbitrary public Internet firewall range was identified for
  PostgreSQL.
- Azure Network Policy capability is enabled.
- Kubernetes authentication remains required for the AKS API.

### Security Concerns

- The application is Internet-facing.
- No Service-level source allowlist was identified.
- The application is reachable over HTTP.
- No working HTTPS endpoint was identified.
- The AKS control plane is not private.
- AKS authorized IP ranges include `0.0.0.0/0`.
- PostgreSQL public network access is enabled.
- The PostgreSQL Azure-services firewall rule creates a broad Azure trust
  boundary.
- No workload NetworkPolicy was identified.

### Preliminary Control Status

**Partially Effective**

Network-security controls exist, but the environment currently contains
several broad/public network paths and does not demonstrate a strong
least-exposure architecture.

---

# SA-03 — Secrets Management

## Security Objective

Determine whether passwords, tokens, database credentials, Terraform
secrets and CI/CD credentials are appropriately separated from source
code and protected from accidental disclosure.

---

## SA-03.1 — Kubernetes Database Secret

### Command

    kubectl describe secret learningsteps-secrets

### Why This Command Was Used

The command was used to inspect the Kubernetes Secret without retrieving
or decoding the secret value.

### Evidence

The Secret existed as:

    Type: Opaque

The Secret contained a key named:

    database-url

The actual value was not displayed during the assessment.

### Observation

The database connection information is stored in a Kubernetes Secret
rather than directly in the application Deployment manifest.

Kubernetes `Opaque` describes a general-purpose Secret type.

The assessment does not treat base64 representation of Kubernetes Secret
data as encryption.

---

## SA-03.2 — Application Secret Consumption

### Command

    kubectl get deployment learningsteps-api \
      -o jsonpath='{range .spec.template.spec.containers[*].env[*]}{.name}{" -> secret="}{.valueFrom.secretKeyRef.name}{", key="}{.valueFrom.secretKeyRef.key}{"\n"}{end}'

### Evidence

The command returned:

    DATABASE_URL -> secret=learningsteps-secrets, key=database-url

### Observation

`DATABASE_URL` is sourced from the Kubernetes Secret and is not
hard-coded into the live Deployment configuration.

---

## SA-03.3 — GRC Repository Credential-File Search

### Command

    find . -type f \( \
      -name ".env" -o \
      -name "*.pem" -o \
      -name "*.key" -o \
      -name "*secret*" -o \
      -name "*credentials*" \
    \) -not -path "./.git/*"

### Evidence

No matching files were returned from the GRC repository.

### Git History Command

    git log --all --name-only --pretty=format: | \
    grep -Ei '(^|/)(\.env|.*\.pem|.*\.key|.*secret.*|.*credentials.*)$' | \
    sort -u

### Evidence

No matching historical filenames were returned.

### Observation

No obvious credential-related files were identified in the current GRC
repository or its Git history using the tested filename patterns.

This does not constitute an exhaustive secret scan.

---

## SA-03.4 — Production Repository Secret Manifest

### Command

    find . -type f \( \
      -name ".env" -o \
      -name "*.pem" -o \
      -name "*.key" -o \
      -name "*secret*" -o \
      -name "*credentials*" \
    \) -not -path "./.git/*"

### Evidence

The production repository contained:

    ./k8s/github-deployer-secret.yaml

### Safe Structure Inspection

    grep -E '^(apiVersion|kind|metadata:|  name:|type:|data:|stringData:)' \
      k8s/github-deployer-secret.yaml

### Evidence

The manifest defined:

    kind: Secret
    name: github-deployer-token
    type: kubernetes.io/service-account-token

No token value was displayed.

---

## SA-03.5 — Git Tracking of Kubernetes Secret Manifest

### Command

    git ls-files --error-unmatch k8s/github-deployer-secret.yaml >/dev/null 2>&1 \
      && echo "TRACKED BY GIT" \
      || echo "NOT TRACKED BY GIT"

### Evidence

The command returned:

    TRACKED BY GIT

### History Command

    git log --all --oneline -- k8s/github-deployer-secret.yaml

### Evidence

The file was present in repository history.

### Secret Data Check

    grep -Eq '^(data|stringData):' k8s/github-deployer-secret.yaml \
      && echo "SECRET DATA FIELD PRESENT" \
      || echo "NO SECRET DATA FIELD PRESENT"

### Evidence

The command returned:

    NO SECRET DATA FIELD PRESENT

### Observation

A Kubernetes service-account-token Secret definition is tracked by Git.

However, no `data:` or `stringData:` field was identified in the
manifest.

The assessment therefore found no evidence that the actual generated
service-account token value was committed in this file.

---

## SA-03.6 — Terraform PostgreSQL Password Handling

### Safe Filename Search

    grep -Rl \
      --exclude-dir=.git \
      --exclude-dir=.terraform \
      "postgresql_admin_password" \
      infra-terraform/

### Evidence

References were identified in:

    infra-terraform/README.md
    infra-terraform/variables.tf
    infra-terraform/postgresql.tf

### Tracked Variable Files

    git ls-files 'infra-terraform/*.tfvars' 'infra-terraform/*.tfvars.json'

### Evidence

The repository tracks:

    infra-terraform/terraform.tfvars

### Password Presence Test

    grep -qE '^[[:space:]]*postgresql_admin_password[[:space:]]*=' \
      infra-terraform/terraform.tfvars \
      && echo "PASSWORD VARIABLE PRESENT IN TRACKED TFVARS" \
      || echo "PASSWORD VARIABLE NOT PRESENT IN TRACKED TFVARS"

### Evidence

The command returned:

    PASSWORD VARIABLE NOT PRESENT IN TRACKED TFVARS

### Environment Variable Name Check

    env | cut -d= -f1 | grep '^TF_VAR_' | sort

### Evidence

The environment included:

    TF_VAR_postgresql_admin_password

No password value was displayed.

### Observation

The PostgreSQL administrator password is not stored in the tracked
`terraform.tfvars` file.

The password is supplied to Terraform through the expected `TF_VAR_`
environment-variable mechanism.

The Terraform variable itself is marked `sensitive = true`.

---

## SA-03.7 — Terraform State and Git

### Git Ignore Evidence

Relevant `.gitignore` entries included:

    *.tfstate
    *.tfstate.*
    .terraform/
    *.tfvars.local
    *.auto.tfvars.local

### Verification Command

    git ls-files | grep -E '(^|/).*\.tfstate(\..*)?$' \
      || echo "NO TERRAFORM STATE FILES TRACKED BY GIT"

### Evidence

The command returned:

    NO TERRAFORM STATE FILES TRACKED BY GIT

### Observation

No Terraform state file was identified as tracked by Git.

This is important because Terraform state may contain sensitive
information even when Terraform variables are marked `sensitive`.

---

## SA-03.8 — Terraform Remote Backend

### Backend Configuration

The Terraform backend was configured as:

    backend "azurerm"

with:

    storage_account_name = "tfstateblob234"
    container_name       = "tfstate"

The state key is stored under the LearningSteps application path.

No storage-account key, SAS token or other credential was identified in
`backend.tf`.

### Observation

Terraform state is designed to use a remote Azure Storage backend rather
than being committed to the source repository.

---

## SA-03.9 — Terraform State Storage Security

### Command

    az storage account show \
      --resource-group containers \
      --name tfstateblob234 \
      --query "{HTTPSOnly:enableHttpsTrafficOnly,PublicNetworkAccess:publicNetworkAccess,MinimumTLS:minimumTlsVersion,SharedKeyAccess:allowSharedKeyAccess,DefaultAction:networkRuleSet.defaultAction}" \
      -o table

### Evidence

The output demonstrated:

    HTTPSOnly: True
    MinimumTLS: TLS1_2
    DefaultAction: Allow

`PublicNetworkAccess` and `SharedKeyAccess` were not displayed in the
table result and were therefore not interpreted as either enabled or
disabled.

### Public Blob Access Check

    az storage account show \
      --resource-group containers \
      --name tfstateblob234 \
      --query "allowBlobPublicAccess" \
      -o tsv

### Evidence

The command returned:

    false

### Container Check

    az storage container show \
      --account-name tfstateblob234 \
      --name tfstate \
      --auth-mode login \
      --query "{Name:name,PublicAccess:properties.publicAccess}" \
      -o table

### Evidence

The container existed and no public-access level was displayed.

### Observation

Positive controls include:

- HTTPS enforcement;
- TLS 1.2 minimum;
- anonymous blob public access disabled;
- use of Microsoft Entra login for the assessment command.

The storage-account network rule default action is `Allow`, meaning the
storage firewall does not provide a default-deny network boundary.

---

## SA-03.10 — GitHub Actions Secret Usage

### Workflow Discovery

    find .github/workflows -type f -maxdepth 1 -print

### Evidence

Two workflows were identified:

    .github/workflows/app-pipeline.yml
    .github/workflows/infra-pipeline.yml

### Secret Reference Search

    grep -RhoE 'secrets\.[A-Za-z0-9_]+' .github/workflows/ 2>/dev/null | \
      sort -u

### Evidence

The workflows reference GitHub Actions Secrets including:

    ARM_CLIENT_ID
    ARM_CLIENT_SECRET
    ARM_SUBSCRIPTION_ID
    ARM_TENANT_ID
    AZURE_CREDENTIALS
    BACKEND_CONTAINER_NAME
    BACKEND_RESOURCE_GROUP
    BACKEND_STORAGE_ACCOUNT
    POSTGRESQL_ADMIN_PASSWORD

No secret values were retrieved.

### Observation

CI/CD secrets are referenced through GitHub Actions Secrets rather than
being directly embedded in the workflow YAML observed during the
assessment.

---

## SA-03.11 — Application Pipeline Authentication

### Evidence

The application workflow declares:

    permissions:
      id-token: write
      contents: read

However, Azure authentication is performed using:

    uses: azure/login@v1

with:

    creds: ${{ secrets.AZURE_CREDENTIALS }}

### Authentication Search

    grep -RnE 'client-id:|tenant-id:|subscription-id:|creds:|id-token:' \
      .github/workflows/

### Evidence

The search showed the OIDC token permission and the `creds:` based Azure
login.

No `client-id`, `tenant-id` and `subscription-id` OIDC login
configuration was identified.

### Observation

Although `id-token: write` is granted, the observed application pipeline
uses the stored `AZURE_CREDENTIALS` secret for Azure authentication.

The evidence therefore does not demonstrate that GitHub-to-Azure OIDC
federation is being used.

---

## SA-03.12 — Infrastructure Pipeline Authentication

### Evidence

The Terraform plan and apply jobs use GitHub Secrets to populate:

    ARM_CLIENT_ID
    ARM_CLIENT_SECRET
    ARM_SUBSCRIPTION_ID
    ARM_TENANT_ID

The PostgreSQL password is supplied using:

    TF_VAR_postgresql_admin_password:
      ${{ secrets.POSTGRESQL_ADMIN_PASSWORD }}

Terraform backend identifiers are also supplied through GitHub Secrets.

The Terraform Apply job declares:

    environment: production

### Observation

Sensitive values are separated from source code through GitHub Actions
Secrets.

However, Azure authentication relies on a stored service-principal client
secret.

This creates a long-lived credential that requires appropriate
protection, expiration and rotation.

The presence of `environment: production` is noted as a potentially
positive deployment-control mechanism, but the workflow alone does not
prove that GitHub Environment approval or protection rules are
configured.

---

## SA-03 Control Assessment

### Positive Controls

- Database connection information is stored in a Kubernetes Secret.
- `DATABASE_URL` references the Secret rather than containing the value.
- No actual service-account token value was identified in the tracked
  Secret manifest.
- PostgreSQL administrator password was not identified in tracked
  `terraform.tfvars`.
- Terraform password input uses a `TF_VAR_` environment variable.
- Terraform password variable is marked sensitive.
- Terraform state is not tracked by Git.
- Terraform uses a remote Azure Storage backend.
- HTTPS is required for the Terraform state storage account.
- Minimum storage TLS version is TLS 1.2.
- Anonymous blob access is disabled.
- CI/CD workflow files reference GitHub Actions Secrets rather than
  observed literal credential values.

### Security Concerns

- Kubernetes secrets remain part of the cluster's credential trust
  boundary.
- CI/CD depends on stored Azure credentials.
- Terraform authentication uses an `ARM_CLIENT_SECRET`.
- The application pipeline uses `AZURE_CREDENTIALS` despite having
  `id-token: write`.
- OIDC/federated Azure authentication was not demonstrated.
- Terraform state storage uses a network default action of `Allow`.
- No centralized application secrets platform such as Azure Key Vault
  was identified during this assessment.

### Preliminary Control Status

**Partially Effective**

Secrets are generally separated from application source code and several
good protections are implemented.

However, long-lived CI/CD credentials and opportunities for stronger
centralized and federated secret management remain.

---

# SA-04 — Kubernetes Workload Security

## Security Objective

Determine whether the LearningSteps Kubernetes workload is configured
to reduce container privilege, unnecessary credential exposure and
workload-to-workload attack paths.

---

## SA-04.1 — Container Security Context

### Command

    kubectl get deployment learningsteps-api \
      -o jsonpath='{range .spec.template.spec.containers[*]}{"Container: "}{.name}{"\nRunAsNonRoot: "}{.securityContext.runAsNonRoot}{"\nRunAsUser: "}{.securityContext.runAsUser}{"\nPrivileged: "}{.securityContext.privileged}{"\nAllowPrivilegeEscalation: "}{.securityContext.allowPrivilegeEscalation}{"\nReadOnlyRootFilesystem: "}{.securityContext.readOnlyRootFilesystem}{"\nCapabilitiesDropped: "}{.securityContext.capabilities.drop}{"\n"}{end}'

### Evidence

The live container returned:

    Container: learningsteps-api
    RunAsNonRoot: true
    RunAsUser: 1000
    Privileged:
    AllowPrivilegeEscalation: false
    ReadOnlyRootFilesystem: false
    CapabilitiesDropped: ["ALL"]

### Observation

Positive workload controls include:

- execution as a non-root user;
- explicit UID 1000;
- privilege escalation disabled;
- all Linux capabilities dropped.

The `Privileged` field was blank and was therefore not interpreted as an
explicitly configured value.

The root filesystem is explicitly writable.

---

## SA-04.2 — Pod Service Account and Host Namespace Configuration

### Command

    kubectl get pods \
      -l app=learningsteps-api \
      -o custom-columns='NAME:.metadata.name,SERVICEACCOUNT:.spec.serviceAccountName,HOSTNETWORK:.spec.hostNetwork,HOSTPID:.spec.hostPID,HOSTIPC:.spec.hostIPC'

### Evidence

Both application pods used:

    SERVICEACCOUNT: default

No host network, host PID or host IPC setting was displayed for the
assessed pods.

### Observation

No evidence of host namespace sharing was identified through this test.

However, the application uses the default Kubernetes service account
rather than a workload-specific service account.

---

## SA-04.3 — Kubernetes Network Policies

### Command

    kubectl get networkpolicies

### Evidence

The command returned:

    No resources found in default namespace.

### Observation

Although Azure Network Policy capability is enabled at cluster level,
no Kubernetes NetworkPolicy resources were identified in the namespace
hosting the LearningSteps workload.

Workload-level network segmentation was therefore not demonstrated.

---

## SA-04.4 — Service Account Token Projection

### Service Account Command

    kubectl get serviceaccount default \
      -o jsonpath='automountServiceAccountToken={.automountServiceAccountToken}{"\n"}'

### Evidence

The value was blank.

A blank value was not interpreted as proof that token mounting was
disabled.

### Pod Verification

    kubectl get pods \
      -l app=learningsteps-api \
      -o jsonpath='{range .items[*]}{.metadata.name}{" | automount="}{.spec.automountServiceAccountToken}{" | token-volume="}{range .spec.volumes[*]}{.projected.sources[*].serviceAccountToken.path}{" "}{end}{"\n"}{end}'

### Evidence

Both LearningSteps pods showed a projected service-account token volume.

### Observation

The application pods receive Kubernetes service-account credentials.

No requirement for the LearningSteps application to communicate directly
with the Kubernetes API was demonstrated during this assessment.

The projected credential therefore represents unnecessary credential
exposure unless an application requirement exists.

---

## SA-04.5 — Deployed Container Image

### Command

    kubectl get deployment learningsteps-api \
      -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{" -> "}{.image}{"\n"}{end}'

### Evidence

The deployed container image used an ACR image tag corresponding to a
Git commit SHA.

The exact repository revision is omitted from this evidence summary.

### Observation

Using a source commit SHA as the deployed image tag provides stronger
deployment traceability than relying on the mutable `latest` tag.

A commit-SHA tag is not equivalent to immutable container digest pinning,
but it provides useful traceability between source code and deployment.

---

## SA-04 Control Assessment

### Positive Controls

- Application runs as non-root.
- Explicit non-root UID is configured.
- Privilege escalation is disabled.
- All Linux capabilities are dropped.
- No host namespace sharing was identified.
- Deployment uses a source commit-SHA image tag.
- Azure Network Policy capability exists at cluster level.

### Security Concerns

- Root filesystem remains writable.
- Application uses the default Kubernetes service account.
- Kubernetes service-account credentials are projected into the
  application pods.
- No workload NetworkPolicy resources were identified.
- A dedicated least-privilege workload identity was not demonstrated.

### Preliminary Control Status

**Partially Effective**

The workload contains several strong container-hardening controls.

However, identity isolation, credential minimization, filesystem
hardening and network segmentation can be improved.

---

# 3. Consolidated Technical Findings

The technical assessment identified the following principal security
concerns.

## F-01 — Broad Azure Privileged Access

Multiple highly privileged human and service-principal identities exist
at subscription or inherited scopes.

**Affected area:** Identity and Access Management

---

## F-02 — Broad Kubernetes Administrative Access

The assessed AKS cluster user has unrestricted Kubernetes permissions
through cluster-level administrative RBAC.

Local AKS accounts also remain enabled.

**Affected area:** Identity and Access Management

---

## F-03 — Public AKS Management Endpoint

The AKS cluster is not private and its authorized API-server IP range
includes `0.0.0.0/0`.

**Affected area:** Network Security

---

## F-04 — Public Application Exposure Without Identified TLS

The LearningSteps API is exposed through an Internet-facing LoadBalancer
on HTTP port 80.

No working HTTPS endpoint was identified on port 443 during the
assessment.

**Affected area:** Network Security

---

## F-05 — PostgreSQL Uses Public Networking

PostgreSQL public network access is enabled.

The identified firewall rule uses the Azure-services access model rather
than restricting access specifically to the LearningSteps workload.

**Affected area:** Network Security

---

## F-06 — Long-Lived CI/CD Azure Credentials

The GitHub Actions workflows use stored Azure credentials, including a
service-principal client secret.

OIDC/federated authentication was not demonstrated.

**Affected area:** Secrets Management / CI/CD

---

## F-07 — Terraform State Network Boundary Can Be Hardened

Terraform state is stored remotely with HTTPS, TLS 1.2 and anonymous
public blob access disabled.

However, the storage network rule default action is `Allow`.

**Affected area:** Secrets Management

---

## F-08 — Default Kubernetes Workload Identity

LearningSteps pods use the Kubernetes `default` service account and
receive projected service-account credentials.

**Affected area:** Kubernetes Workload Security

---

## F-09 — No Workload NetworkPolicy

Azure Network Policy capability is enabled, but no NetworkPolicy
resources were identified for the LearningSteps workload.

**Affected area:** Kubernetes Workload Security / Network Security

---

## F-10 — Writable Container Root Filesystem

The LearningSteps container has:

    readOnlyRootFilesystem: false

Other container-hardening controls are present.

**Affected area:** Kubernetes Workload Security

---

# 4. Positive Security Controls Identified

The assessment also identified important security controls already
implemented in LearningSteps.

These include:

- Azure RBAC;
- Kubernetes RBAC;
- non-root container execution;
- explicit non-root UID;
- privilege escalation disabled;
- all Linux capabilities dropped;
- Azure Network Policy capability enabled;
- Azure Standard Load Balancer;
- PostgreSQL firewall controls;
- Kubernetes Secret use for the database connection;
- no database password identified in tracked `terraform.tfvars`;
- Terraform sensitive variable handling;
- Terraform state excluded from Git;
- remote Azure Storage Terraform backend;
- HTTPS enforcement for Terraform state storage;
- TLS 1.2 minimum for Terraform state storage;
- anonymous blob public access disabled;
- GitHub Actions Secrets used for CI/CD sensitive values;
- source commit-SHA container image deployment;
- infrastructure security scanning using Trivy and tfsec was observed in
  the Terraform workflow.

These controls will be considered when determining residual risk and
ISO/IEC 27001 control effectiveness.

---

# 5. Assessment Limitations

This assessment was intentionally scoped as a GRC-supporting technical
security assessment.

It did not include:

- exploitation of identified weaknesses;
- password cracking;
- vulnerability exploitation;
- network port scanning;
- web application penetration testing;
- SQL injection testing;
- authentication bypass attempts;
- privilege-escalation exploitation;
- container escape testing;
- adversarial Kubernetes API testing;
- denial-of-service testing;
- social engineering.

No conclusion should therefore be interpreted as proof that an
unassessed vulnerability does or does not exist.

A separate penetration-testing project may be conducted to evaluate
exploitability.

---

# 6. Next GRC Phase

The technical evidence collected in SA-01 through SA-04 will now be used
to perform the formal GRC analysis.

The next activities are:

1. update the asset inventory where necessary;
2. refine the threat model using the discovered architecture;
3. map technical findings to threats;
4. determine evidence-based likelihood;
5. determine business/security impact;
6. calculate risk ratings;
7. map applicable ISO/IEC 27001 controls;
8. perform control-gap analysis;
9. develop prioritized remediation recommendations; and
10. develop supporting security policies.

The distinction between technical evidence and risk analysis will be
maintained throughout the remainder of the project.
