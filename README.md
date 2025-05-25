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

## Pipeline Structure

Initially, I set up a simple CI/CD pipeline using standard dotnet build and ZIP-based artefacts deployment, just to validate that the build worked and the app could be deployed to Azure App Service.  
After that I moved to a Docker based multi-stage pipeline for better control and consistency, this will help everyone to deploy the image to any environment without any dependency issue. This approach allowed me to containerize the app cleanly, optimize build layers, and push images to ACR as artifacts.  
I structured the pipeline into three CI stages:  
**Test**: Run unit tests early to fail fast  
**Build**: Compile and restore dependencies  
**Publish**: Docker build and push to ACR  

Then I separated out CD as a single deployment stage, which deploys the Docker image to Azure App Service.  

---

## Azure Container Registry

Initially considered Docker Hub, but switched to **Azure Container Registry (ACR)** because of Seamless integration with Azure App Service through service connections it is easier and more secure setup without managing access tokens

---

## Azure Repos

I had the option to choose GitHub for managing my repository but I chose Azure Repos to keep everything within the same ecosystem. So  managing code, pipelines, and policies were easier as all was in one place. Also I had the option to write the YAML in the Azure devops to test my ci and cd triggers. 

---

## Branch Strategy

I have created branch polices to protected the master branch by disabling direct commits. I created a dev branch for all working changes and enabled a pull request workflow. CI runs automatically on PRs, and once the build passes, the PR can be approved and merged. This will have more control over unnecessary changes and make the master branch safe.

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
