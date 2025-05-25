# Docosoft .NET Application Deployment Assignment

This project demonstrates an end-to-end DevOps workflow that includes:

* CI/CD pipeline using **Azure DevOps Pipelines**
* Containerization of the **.NET 8 Web API using Docker**
* Hosting through **Azure App Service**
* Source control maintained through **Azure Repos**
* Docker container registry used: **Azure Container Registry (ACR)**
* Deployment security through **branch protection and PR workflows**

---

## Application URL

**Live Endpoint:** [https://docosoft-counter-app-asgbfthybzfwafg0.westeurope-01.azurewebsites.net/count](https://docosoft-counter-app-asgbfthybzfwafg0.westeurope-01.azurewebsites.net/count)


![Ci Pipeline](./screenshots/webapp.jpg)
---

## Azure repository Structure

```bash
.
├── src/                      # Source code of the .NET Web API
├── tests/                    # Unit test project
├── azure-build.yml           # CI pipeline YAML definition
├── azure-release.yml         # CD pipeline YAML definition
└── Dockerfile                # Multi-stage Dockerfile for the application
```

---

## High-Level Architecture

![CI/CD Architecture](./screenshots/architecture.jpg)

---

## Design Decisions and Thought Process Behind the Project

### Pipeline Structure

Initially, I set up a simple CI/CD pipeline using standard `dotnet build` and ZIP-based artefacts deployment, just to validate that the build worked and the app could be deployed to Azure App Service.
After that, I moved to a Docker-based multi-stage pipeline for better control and consistency, allowing deployment to any environment without dependency issues.
The pipeline has three CI stages:

* **Test**: Run unit tests
* **Build**: Compile and restore dependencies
* **Publish**: Docker build and push to ACR

Then I separated out CD as a single deployment stage, which deploys the Docker image to Azure App Service.

---

### Azure Container Registry

I chose **Azure Container Registry (ACR)** over Docker Hub because of its seamless integration with Azure App Service through service connections. It also provides a more secure setup without the need to manage access tokens.

---

### Azure Repos

I used Azure Repos instead of GitHub to keep the entire workflow within the Azure DevOps ecosystem. This made it easier to manage code, pipelines, and policies all in one place, and also allowed direct YAML authoring within the portal.

---

### Branch Strategy

I implemented branch policies to protect the `master` branch by disabling direct commits. All working changes are made in a `dev` branch. Pull requests trigger CI automatically, and once builds pass, the PR can be approved and merged. This ensures the `master` branch stays stable.

---

## Azure Portal Infrastructure Setup

### Resource Group

I created a resource group named `docosoft-assignment` in the West Europe region. It groups all resources like App Service and ACR for centralized management.

![Resource Group](./screenshots/resource.jpg)

---

### App Service

I deployed an App Service named `docosoft-counter-app`, running on Linux with a B3 plan (3.5 GB RAM). Docker container was used for deployment, and system-assigned managed identity was enabled.

![App Service](./screenshots/appservice.jpg)

---

### Azure Container Registry (ACR)

I created a container registry named `docosoftcounter` in the same region. It stores Docker images built by the CI pipeline. I assigned the AcrPull role to the App Service’s managed identity for secure image pulls.

![Azure Container Registry](./screenshots/container.jpg)

---

## Azure DevOps Setup

### Repository Setup

I used the `docosoft-assignment` repository under the `docosoft-api-project` in Azure Repos. The repo contains:

* .NET source code
* `azure-build.yml` (CI pipeline)
* `azure-release.yml` (CD pipeline)


---

### CI Pipeline – `azure-build.yml`

The CI pipeline runs on every push to `master`. It restores dependencies, runs tests, builds the .NET app, and creates/pushes the Docker image to ACR using a multi-stage Dockerfile.

```yaml
trigger:
  branches:
    include:
      - master  # Trigger CI pipeline when changes are pushed to master

variables:
  vmImageName: 'ubuntu-latest'
  dockerRegistryServiceConnection: '8a597463-f084-4d63-9c64-6a540d53b1ec'
  imageRepository: 'counterapi'
  containerRegistry: 'docosoftcounter.azurecr.io'
  dockerfilePath: '**/Dockerfile'
  tag: '$(Build.BuildId)'

stages:
- stage: Test
  displayName: 'Test the Application'
  jobs:
  - job: RunTests
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: UseDotNet@2
      inputs:
        packageType: 'sdk'
        version: '8.0.x'
    - script: dotnet restore CounterApi.sln
      displayName: 'Restore Dependencies'
    - script: dotnet test CounterApi.sln --configuration Release --verbosity normal
      displayName: 'Run Unit Tests'

- stage: Build
  displayName: 'Build the Application'
  dependsOn: Test
  jobs:
  - job: BuildApp
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: UseDotNet@2
      inputs:
        packageType: 'sdk'
        version: '8.0.x'
    - script: dotnet build CounterApi.sln --configuration Release
      displayName: 'Build Solution'

- stage: Publish
  displayName: 'Publish via Docker to ACR'
  dependsOn: Build
  jobs:
  - job: DockerPush
    pool:
      vmImage: $(vmImageName)
    steps:
    - checkout: self
    - task: Docker@2
      displayName: 'Build and Push Docker Image'
      inputs:
        command: buildAndPush
        containerRegistry: $(dockerRegistryServiceConnection)
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        buildContext: '$(Build.SourcesDirectory)'
        tags: |
          $(tag)
          latest
```

It restores dependencies, runs tests, builds the .NET app, and creates/pushes the Docker image to ACR using a multi-stage Dockerfile.

![Ci Pipeline](./screenshots/ci.jpg)


---

### CD Pipeline – `azure-release.yml`

The CD pipeline deploys the image from ACR to Azure App Service. It’s triggered either automatically after a successful CI build.

```yaml
trigger: none  # Triggers after CI pipeline succeeds

resources:
  pipelines:
    - pipeline: buildPipeline
      source: azure-build
      trigger:
        branches:
          include:
            - master

pool:
  vmImage: 'ubuntu-latest'

variables:
  webAppName: 'docosoft-counter-app'
  imageName: 'counterapi'
  acrLoginServer: 'docosoftcounter.azurecr.io'
  dockerTag: 'latest'

steps:
- task: AzureWebAppContainer@1
  displayName: 'Deploy Docker Image to Azure App Service'
  inputs:
    azureSubscription: 'sunandan'
    appName: '$(webAppName)'
    containers: '$(acrLoginServer)/$(imageName):$(dockerTag)'
```



![CD Pipeline](./screenshots/cd.jpg)

---

### Service Connections

I created two service connections in Azure DevOps:

* Azure Resource Manager (`sunandan`) for App Service deployments
* Docker registry (to ACR) using managed identity for secure image operations

---

### Dockerfile

The Dockerfile uses a multi-stage build:

* Stage 1: Compiles and publishes the .NET app
* Stage 2: Builds a clean runtime image for App Service

```dockerfile
# ---------- Build Stage ----------
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

WORKDIR /app

# Copy solution and all source folders
COPY CounterApi.sln ./
COPY src/ ./src/
COPY tests/ ./tests/

# Restore dependencies
RUN dotnet restore

# Build and publish the app
RUN dotnet publish src/CounterApi.csproj -c Release -o /app/publish

# ---------- Runtime Stage ----------
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime

WORKDIR /app

COPY --from=build /app/publish .

EXPOSE 80

ENTRYPOINT ["dotnet", "CounterApi.dll"]
```

* Stage 1: Compiles and publishes the .NET app
* Stage 2: Builds a clean runtime image for App Service

---

## PR Workflow and Branch Policies

### Branch Policy

* Direct commits to `master` are blocked
* All changes go through pull requests
* PR creator can approve if CI passes

---


### PR Workflow Steps

1. User Pushed code to `dev` branch
2. Created PR → `dev` to `master`
3. CI pipeline automatically triggered:

   * Restore, Build, Test, Docker Push
4. After success → PR approved
5. Merge to `master` → CI triggered again
6. CD pipeline deployed image to App Service
7. App was accessible via live URL

![PR](./screenshots/pr.jpg)


---
