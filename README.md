# LearningSteps GRC Security Assessment

## Project Overview

This project is a Governance, Risk, and Compliance (GRC) security assessment of the LearningSteps application and its supporting Azure cloud infrastructure.

The LearningSteps technical environment was previously designed and deployed as a working cloud-native application. It includes a FastAPI application, PostgreSQL database, Azure Kubernetes Service (AKS), Azure Container Registry (ACR), Terraform infrastructure-as-code, Kubernetes resources, and GitHub Actions CI/CD pipelines.

This GRC project does not rebuild the application. Instead, it evaluates the existing LearningSteps environment from a security, risk, governance, and compliance perspective.

The purpose of the assessment is to answer the management question:

> **Is the LearningSteps environment secure enough, what are its key risks and control gaps, and can the organization provide evidence of its security controls?**

## Assessment Objectives

The project will:

- Define the system and security assessment scope.
- Identify and classify important assets.
- Identify realistic threats affecting the LearningSteps environment.
- Assess security risks based on likelihood and impact.
- Map existing technical and administrative controls to ISO 27001.
- Identify security and compliance gaps.
- Recommend prioritized security improvements.
- Develop practical security policies.
- Collect evidence supporting the assessment findings.

## System Being Assessed

The assessment is based on the deployed LearningSteps technical environment, including:

- FastAPI application
- PostgreSQL database
- Azure Kubernetes Service (AKS)
- Azure Container Registry (ACR)
- Azure networking and public LoadBalancer
- Kubernetes workloads and secrets
- Terraform infrastructure-as-code
- GitHub source control
- GitHub Actions CI/CD pipelines
- Application health and monitoring capabilities

The technical implementation is maintained separately from this GRC assessment repository.

## Assessment Framework

ISO/IEC 27001 will be used as the primary framework for control mapping.

Where useful, the assessment may also reference security principles and practices associated with cloud security, secure software development, access control, monitoring, vulnerability management, and risk management.

## Repository Structure

- `01-system-scope/` — Assessment boundaries, assumptions, and exclusions
- `02-asset-inventory/` — Identification and classification of assets
- `03-threat-model/` — Threats relevant to the deployed architecture
- `04-risk-assessment/` — Likelihood, impact, and risk evaluation
- `05-control-mapping/` — Mapping existing controls to ISO 27001
- `06-gap-analysis/` — Missing, weak, or insufficient controls
- `07-recommendations/` — Prioritized remediation actions
- `08-policies/` — Security and governance policies
- `evidence/` — Technical evidence supporting assessment conclusions

## Assessment Approach

The assessment follows the sequence:

**Scope → Assets → Threats → Risks → Controls → Gaps → Recommendations → Policies → Evidence**

The findings will be based on the actual LearningSteps implementation rather than a hypothetical architecture.

## Project Status

**Technical prerequisite:** Completed and operational.

**GRC assessment:** In progress.
