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

Azure Portal infrastructure setup
Resource Group:
I created a resource group named docosoft-assignment in the West Europe region. This served as a container for managing all related resources such as the App Service and Azure Container Registry in a centralized and organized way.

App Service:
I deployed an App Service named docosoft-counter-app, running on Linux in the West Europe region. For the hosting plan, I selected the Basic B3 tier, which offers 3.5 GB of memory—sufficient for stable containerized .NET application deployment.

Azure Container Registry (ACR):
I created a container registry named docosoftcounter in the same region. This registry stores Docker images built by the CI pipeline. I enabled system-assigned managed identity on the App Service and assigned the AcrPull role to it in ACR. This allows secure and seamless image pulls without storing credentials.
---

## Azure DevOps Setup

### 🔹 Repository Setup:

I used the `docosoft-assignment` repository under the `docosoft-api-project` in Azure Repos. The repo contains the .NET application code along with two YAML files: `azure-build.yml` for CI and `azure-release.yml` for CD. Keeping everything in one place ensured easy version control and seamless pipeline integration.

### 🔹 CI Pipeline – `azure-build.yml`:

I created the `azure-build.yml` file to define my CI pipeline. It runs automatically whenever I push changes to the `master` branch. I used the Microsoft-hosted `ubuntu-latest` agent and structured the pipeline into three stages — test, build, and publish. First, it restores dependencies and runs unit tests. Then it builds the .NET application and finally uses a multi-stage Dockerfile to build and push the Docker image to my ACR (`docosoftcounter`).

### 🔹 CD Pipeline – `azure-release.yml`:

To handle deployments, I created a separate CD pipeline in `azure-release.yml`. This pipeline either runs manually or is triggered after a successful CI build. It pulls the latest image from ACR and deploys it to my Azure App Service (`docosoft-counter-app`) using the `AzureWebAppContainer@1` task. Keeping CI and CD separate made it easier for me to test and debug each part independently.

### 🔹 Service Connections:

In Azure DevOps, I set up two service connections to securely integrate with my Azure resources. The first one is an Azure Resource Manager connection named `sunandan`, which I used to manage deployments to App Service. The second one connects to ACR and uses managed identity, so I didn’t have to deal with storing credentials or access tokens.

### 🔹 Dockerfile:

I wrote a multi-stage Dockerfile to optimize the build process and reduce the image size. In the first stage, I compile and publish the .NET application. In the second stage, I copy the published output into a lightweight runtime image that’s used for deployment on Azure App Service.

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
