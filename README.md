# Docosoft .NET Application Deployment Assignment

This project demonstrates an end-to-end DevOps workflow that includes:

- CI/CD pipeline using **Azure DevOps Pipelines**  
- Containerization of the **.NET 8 Web API using Docker**  
- Hosting through **Azure App Service**  
- Source control maintained through **Azure Repos**  
- Docker container registry used: **Azure Container Registry (ACR)**  
- Deployment security through **branch protection and PR workflows**

---

## Design decisions and Thought Process Behind the Project

### Pipeline Structure

- Initially, I set up a simple CI/CD pipeline using standard `dotnet build` and **ZIP-based artefact deployment**, just to validate that the build worked and the app could be deployed to Azure App Service.
- After that, I moved to a **Docker-based multi-stage pipeline** for better control and consistency, helping anyone deploy the image to any environment without dependency issues.
- This approach allowed me to containerize the app cleanly, optimize build layers, and push images to ACR as artifacts.

### CI Pipeline Stages

1. **Test** – Run unit tests early to fail fast  
2. **Build** – Compile and restore dependencies  
3. **Publish** – Docker build and push to ACR

> I separated out CD as a single deployment stage, which deploys the Docker image to Azure App Service.

---

## Azure Container Registry

- Initially considered Docker Hub, but switched to **Azure Container Registry (ACR)** because:
  - Seamless integration with Azure App Service via service connections
  - Easier and more secure setup without managing access tokens

---

## Azure Repos

- I chose Azure Repos over GitHub to keep everything within the same ecosystem.
- Managing code, pipelines, and policies became simpler in one place.
- Also enabled YAML authoring/testing directly inside Azure DevOps.

---

## Branch Strategy

- **Branch protection**: Master branch is protected by disabling direct commits.
- **Working changes** happen in a `dev` branch.
- **Pull request (PR) workflow** enabled:
  - CI runs automatically on PRs
  - Once CI passes, PR can be approved and merged

This adds better control over changes and ensures the master branch remains stable.

---

## Infrastructure Setup using Azure Portal

### 1. Resource Group Creation

- Created **Resource Group**: `docosoft-assignment`
- Region: **West Europe**
- Purpose: Group related resources for better management and cost tracking

---

### 2. App Service Configuration

- **App Service Name**: `docosoft-counter-app`
- OS: **Linux**
- Plan: **Basic B1 (1.75 GB RAM)**
- Region: **West Europe**
- Publishing method: **Docker Container**

Enabled **System-assigned Managed Identity** for secure ACR authentication (no Docker Hub creds or manual secrets needed).

---

### 3. Azure Container Registry (ACR)

- Name: `docosoftcounter`
- Region: **West Europe**
- Placed in same resource group for performance and simplicity
- Assigned **AcrPull** role to App Service’s managed identity to securely pull images

---

## Azure DevOps Setup (CI/CD Implementation)

Automated build, test, and deployment using Azure DevOps with YAML pipelines.

---

### 1. Repository Setup

- **DevOps Project**: `docosoft-api-project`  
- **Repository**: `docosoft-assignment`  
- Contains:
  - .NET application code  
  - `azure-build.yml` (CI pipeline)  
  - `azure-release.yml` (CD pipeline)  

---

### 2. CI Pipeline – `azure-build.yml`

**Trigger**: On push to `master`  
**Agent Pool**: `ubuntu-latest`  

**Stages**:

- **Test**: Restore dependencies, run unit tests  
- **Build**: Compile the .NET app  
- **Publish**: Multi-stage Docker build + push to ACR (`docosoftcounter`)

Ensures that every push to `master` runs full build + Docker image update.

---

### 3. CD Pipeline – `azure-release.yml`

**Trigger**: After successful CI or manually

**Steps**:

- Pull latest Docker image from ACR  
- Deploy to **Azure App Service** (`docosoft-counter-app`) using `AzureWebAppContainer@1`

Separation of CI/CD simplifies debugging and makes maintenance easier.

---

### 4. Service Connections

#### Azure Resource Manager:

- **Name**: `sunandan`  
- Used for provisioning and deployment to App Service

#### Azure Container Registry (Docker):

- Used for ACR authentication  
- Secured using **Managed Identity**

---

### 5. Dockerfile

- Multi-stage Dockerfile:
  - Stage 1: Compile and publish app
  - Stage 2: Build lightweight runtime image for App Service

---

## Pull Request Workflow & Branch Policy

Ensures safe, auditable changes to the production `master` branch.

---

### Branch Policy

- Blocked direct commits to `master`
- Feature/Dev branches must raise a **PR**
- PR creator is allowed to approve if build passes

---

### PR Workflow Steps

1. Created a `dev` branch → pushed changes  
2. Raised PR → `dev` to `master`  
3. CI pipeline ran automatically:  
   - Restore, Build, Test, Docker Push  
4. If successful → PR approved  
5. On merge → CI re-ran on `master`  
6. CD pipeline deployed latest Docker image to App Service  
7. Live URL confirmed successful deployment

---
