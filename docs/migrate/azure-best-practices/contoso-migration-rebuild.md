---
title: Rebuild an on-premises application to Azure
description: Learn how Contoso rebuilds an application in Azure using Azure App Service, Kubernetes Service, Cosmos DB, Functions, and Cognitive Services.
author: BrianBlanchard
ms.author: brblanch
ms.date: 7/1/2020
ms.topic: conceptual
ms.service: cloud-adoption-framework
ms.subservice: migrate
services: site-recovery
---

<!-- docsTest:ignore "Enable .NET" SmartHotel360 SmartHotel360-Backend Pet.Checker contoso-datacenter git aks PetCheckerFunction -->

<!-- cSpell:ignore givenscj SQLVM WEBVM contosohost vcenter contosodc smarthotel contososmarthotel smarthotelcontoso smarthotelpetchecker petchecker smarthotelakseus smarthotelacreus smarthotelpets kubectl contosodevops visualstudio azuredeploy cloudapp smarthotelsettingsurl appsettings -->

# Rebuild an on-premises application in Azure

This article demonstrates how the fictional company Contoso rebuilds a two-tier Windows .NET application running on VMware VMs as part of a migration to Azure. Contoso migrates the front-end VM to an Azure App Service web app. The application back end is built using microservices deployed to containers managed by Azure Kubernetes Service (AKS). The site interacts with Azure Functions to provide pet photo functionality.

The SmartHotel360 application used in this example is provided as open source. If you'd like to use it for your own testing purposes, you can download it from [GitHub](https://github.com/Microsoft/SmartHotel360).

## Business drivers

The IT leadership team has worked closely with business partners to understand what they want to achieve with this migration:

- **Address business growth.** Contoso is growing and wants to provide differentiated experiences for customers on Contoso websites.
- **Be agile.** Contoso must be able to react faster than the changes in the marketplace to enable the success in a global economy.
- **Scale.** As the business grows successfully, the Contoso IT team must provide systems that can grow at the same pace.
- **Reduce costs.** Contoso wants to minimize licensing costs.

## Migration goals

The Contoso cloud team has pinned down application requirements for this migration. These requirements were used to determine the best migration method:

- The application in Azure is still as critical as it is today. It should perform well and scale easily.
- The application shouldn't use IaaS components. Everything should be built to use PaaS or serverless services.
- Application builds should run in cloud services, and containers should reside in a private enterprise-wide registry in the cloud.
- The API service used for pet photos should be accurate and reliable in the real world, since decisions made by the application must be honored in their hotels. Any pet granted access is allowed to stay at the hotels.
- To meet requirements for a DevOps pipeline, Contoso will use a Git repository in Azure Repos for source code management. Automated builds and releases will be used to build code and deploy to Azure App Service, Azure Functions, and AKS.
- Different continuous integration/continuous development (CI/CD) pipelines are needed for microservices on the back end and for the web site on the front end.
- The back-end services have a different release cycle from the front-end web app. To meet this requirement, they will deploy two different pipelines.
- Contoso needs management approval for all front-end website deployment, and the CI/CD pipeline must provide this.

## Solution design

After pinning down goals and requirements, Contoso designs and review a deployment solution, and identifies the migration process, including the Azure services that will be used for the migration.

### Current application

- The SmartHotel360 on-premises application is tiered across two VMs (`WEBVM` and `SQLVM`).
- The VMs are located on VMware ESXi host `contosohost1.contoso.com` (version 6.5).
- The VMware environment is managed by vCenter Server 6.5 (`vcenter.contoso.com`), running on a VM.
- Contoso has an on-premises datacenter (`contoso-datacenter`), with an on-premises domain controller (`contosodc1`).
- The on-premises VMs in the Contoso datacenter will be decommissioned after the migration is done.

### Proposed architecture

- The front end of the application is deployed as an Azure App Service web app in the primary Azure region.
- An Azure function provides uploads of pet photos, and the site interacts with this functionality.
- The pet photo function uses the Computer Vision API of Azure Cognitive Services along with Azure Cosmos DB.
- The back end of the site is built using microservices. These will be deployed to containers managed in AKS.
- Containers will be built using Azure DevOps, and pushed to Azure Container Registry.
- For now, Contoso will manually deploy the web app and function code using Visual Studio.
- Microservices will be deployed using a PowerShell script that calls Kubernetes command-line tools.

    ![Scenario architecture](./media/contoso-migration-rebuild/architecture.png)
    _Figure 1: Scenario architecture._

### Solution review

Contoso evaluates the proposed design by putting together a pros and cons list.

| Consideration | Details |
| --- | --- |
| **Pros** | Using PaaS and serverless solutions for the end-to-end deployment significantly reduces management time that Contoso must provide. <br><br> Moving to a microservices-based architecture allows Contoso to easily extend the solution over time. <br><br> New functionality can be brought online without disrupting any of the existing solutions code bases. <br><br> The web app will be configured with multiple instances with no single point of failure. <br><br> Autoscaling will be enabled so that the application can handle differing traffic volumes. <br><br> With the move to PaaS services, Contoso can retire out-of-date solutions running on Windows Server 2008 R2 operating system. <br><br> Azure Cosmos DB has built-in fault tolerance, which requires no configuration by Contoso. This means that the data tier is no longer a single point of failover. |
| **Cons** | Containers are more complex than other migration options. The learning curve could be an issue for Contoso. They introduce a new level of complexity that provides value in spite of the curve. <br><br> The operations team at Contoso needs to ramp up to understand and support Azure, containers, and microservices for the application. <br><br> Contoso hasn't fully implemented DevOps for the entire solution. Contoso needs to consider that for the deployment of services to AKS, Azure Functions, and Azure App Service. |

### Migration process

1. Contoso provisions Azure Container Registry, AKS, and Azure Cosmos DB.
2. Contoso provisions the infrastructure for the deployment, including Azure App Service web app, storage account, function, and API.
3. After the infrastructure is in place, they'll build their microservices container images using Azure DevOps, which pushes them to the container registry.
4. Contoso will deploy these microservices to AKS using a PowerShell script.
5. Finally, they'll deploy the function and web app.

    ![Migration process](./media/contoso-migration-rebuild/migration-process.png)
    _Figure 2: The migration process._

### Azure services

| Service | Description | Cost |
|---|---|---|
| [AKS](https://docs.microsoft.com/sql/dma/dma-overview?view=ssdt-18vs2017) | Simplifies Kubernetes management, deployment, and operations. Provides a fully managed Kubernetes container orchestration service. | AKS is a free service. Pay for only the VMs and the associated storage and networking resources consumed. [Learn more](https://azure.microsoft.com/pricing/details/kubernetes-service). |
| [Azure Functions](https://azure.microsoft.com/services/functions) | Accelerates development with an event-driven, serverless compute experience. Scale on demand. | Pay only for consumed resources. Plan is billed based on per-second resource consumption and executions. [Learn more](https://azure.microsoft.com/pricing/details/functions). |
| [Azure Container Registry](https://azure.microsoft.com/services/container-registry) | Stores images for all types of container deployments. | Cost based on features, storage, and usage duration. [Learn more](https://azure.microsoft.com/pricing/details/container-registry). |
| [Azure App Service](https://azure.microsoft.com/services/app-service/containers) | Quickly build, deploy, and scale enterprise-grade web, mobile, and API apps running on any platform. | App Service plans are billed on a per-second basis. [Learn more](https://azure.microsoft.com/pricing/details/app-service/windows). |

## Prerequisites

Here's what Contoso needs for this scenario:

| Requirements | Details |
| --- | --- |
| Azure subscription | <li> Contoso created subscriptions during an earlier article. If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free). <li> If you create a free account, you're the admin of your subscription and can perform all actions. <li> If you use an existing subscription and you're not the admin, then you need to work with the admin to assign Owner or Contributor permissions to you. |
| Azure infrastructure | <li> Learn [how Contoso set up an Azure infrastructure](./contoso-migration-infrastructure.md). |
| Developer prerequisites | Contoso needs the following tools on a developer workstation: <li>  [Visual Studio Community 2017: Version 15.5](https://visualstudio.microsoft.com) <li> Enable .NET workload. <li> [Git](https://git-scm.com) <li> [Azure PowerShell](https://azure.microsoft.com/downloads) <li> [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest) <li> [Docker Community Edition (Windows 10) or Docker Enterprise Edition (Windows Server)](https://docs.docker.com/docker-for-windows/install) set to use Windows containers. |

## Scenario steps

Here's how Contoso will run the migration:

> [!div class="checklist"]
>
> - **Step 1: Provision AKS and Azure Container Registry.** Contoso provisions the managed AKS cluster and the container registry using PowerShell.
> - **Step 2: Build Docker containers.** They set up continuous integration (CI) for Docker containers using Azure DevOps and push them to the container registry.
> - **Step 3: Deploy back-end microservices.** They deploy the rest of the infrastructure that will be used by back-end microservices.
> - **Step 4: Deploy front-end infrastructure.** They deploy the front-end infrastructure, including Blob storage for the pet phones, the Azure Cosmos DB, and Computer Vision API.
> - **Step 5: Migrate the back end.** They deploy microservices and run on AKS to migrate the back end.
> - **Step 6: Publish the front end.** They publish the SmartHotel360 application to Azure App Service along with the function app to be called by the pet service.

## Step 1: Provision back-end resources

Contoso admins run a deployment script to create the managed Kubernetes cluster using AKS and Azure Container Registry. The instructions for this section use the [SmartHotel360-Backend](https://github.com/Microsoft/SmartHotel360-Backend) GitHub repository. The repository contains all of the software for this part of the deployment.

### Ensure prerequisites

Before they start, Contoso admins ensure that all prerequisite software in installed on the dev machine they're using for the deployment. They clone the repository local to the dev machine using Git:

`git clone https://github.com/Microsoft/SmartHotel360-Backend.git`

### Provision AKS and Azure Container Registry

The Contoso admins provision as follows:

1. They open the folder using Visual Studio Code and move to the `/deploy/k8s` directory, which contains the script `gen-aks-env.ps1`.

2. They run the script to create the managed Kubernetes cluster, using AKS and Azure Container Registry.

    ![AKS](./media/contoso-migration-rebuild/aks1.png)
    _Figure 3: Creating the managed Kubernetes cluster._

3. With the file open, they update the $location parameter to `eastus2`, and save the file.

    ![AKS](./media/contoso-migration-rebuild/aks2.png)
    _Figure 4: Saving the file._

4. They select **View** > **Terminal** to open the integrated terminal in Visual Studio Code.

    ![AKS](./media/contoso-migration-rebuild/aks3.png)
    _Figure 5: The terminal in Visual Studio Code._

5. In the PowerShell integrated terminal, they sign into Azure using the `Connect-AzureRmAccount` command. For more information, see [Get started with PowerShell](https://docs.microsoft.com/powershell/azure/get-started-azureps).

    ![AKS](./media/contoso-migration-rebuild/aks4.png)
    _Figure 6: The PowerShell integrated terminal._

6. They authenticate Azure CLI by running the `az login` command and following the instructions to authenticate using their web browser. [Learn more](https://docs.microsoft.com/cli/azure/authenticate-azure-cli?view=azure-cli-latest) about logging in with Azure CLI.

    ![AKS](./media/contoso-migration-rebuild/aks5.png)
    _Figure 7: Authenticating Azure CLI._

7. They run the following command while passing the resource group name of `ContosoRG`, the name of the AKS cluster `smarthotel-aks-eus2`, and the new registry name.

    `.\gen-aks-env.ps1  -resourceGroupName ContosoRg -orchestratorName smarthotelakseus2 -registryName smarthotelacreus2`

    ![AKS](./media/contoso-migration-rebuild/aks6.png)
    _Figure 8: Running the command._

8. Azure creates another resource group that contains the resources for the AKS cluster.

    ![AKS](./media/contoso-migration-rebuild/aks7.png)
    _Figure 9: Azure creating a resource group._

9. After the deployment is finished, they install the `kubectl` command-line tool. The tool is already installed on the Azure Cloud Shell.

    `az aks install-cli`

10. They verify the connection to the cluster by running the `kubectl get nodes` command. The node is the same name as the VM in the automatically created resource group.

    ![AKS](./media/contoso-migration-rebuild/aks8.png)
    _Figure 10: Verifying the connection to the cluster._

11. They run the following command to start the Kubernetes dashboard:

    `az aks browse --resource-group ContosoRG --name smarthotelakseus2`

12. A browser tab opens to the dashboard. This is a tunneled connection using the Azure CLI.

    ![AKS](./media/contoso-migration-rebuild/aks9.png)
    _Figure 11: A tunneled connection._

## Step 2: Configure the back-end pipeline

### Create an Azure DevOps project and build

Contoso creates an Azure DevOps project, and configures a CI build to create the container and then pushes it to the container registry. The instructions in this section use the [SmartHotel360-Backend](https://github.com/Microsoft/SmartHotel360-Backend) repository.

1. From `visualstudio.com`, they create a new organization (`contosodevops360.visualstudio.com`) and configure it to use Git.

2. They create a new project (`SmartHotelBackend`), selecting **Git** for version control and **Agile** for the workflow.

    ![Azure DevOps](./media/contoso-migration-rebuild/vsts1.png)
    _Figure 12: Creating a new project._

3. They import the [GitHub repo](https://github.com/Microsoft/SmartHotel360-Backend).

    ![Azure DevOps](./media/contoso-migration-rebuild/vsts2.png)
    _Figure 13: Importing the GitHub repo._

4. In **Pipelines**, they select **Build** and create a new pipeline using Azure Repos Git as a source from the repository.

    ![Azure DevOps](./media/contoso-migration-rebuild/vsts3.png)
    _Figure 14: Creating a new pipeline._

5. They select to start with an empty job.

    ![Azure DevOps](./media/contoso-migration-rebuild/vsts4.png)
    _Figure 15: Starting with an empty job._

6. They select **Hosted Linux Preview** for the build pipeline.

    ![Azure DevOps](./media/contoso-migration-rebuild/vsts5.png)
    _Figure 16: Setting up the build pipeline._

7. In **Phase 1**, they add a **Docker Compose** task. This task builds the Docker Compose.

    ![Azure DevOps](./media/contoso-migration-rebuild/vsts6.png)
    _Figure 17: Building the Docker Compose._

8. They repeat and add another **Docker Compose** task. This one pushes the containers to the container registry.

     ![Azure DevOps](./media/contoso-migration-rebuild/vsts7.png)
    _Figure 18: Adding another Docker Compose task._

9. They select the first task to build and configure the build with the Azure subscription, authorization, and the container registry.

    ![Azure DevOps](./media/contoso-migration-rebuild/vsts8.png)
    _Figure 19: Building and configuring the build._

10. They specify the path of the `docker-compose.yaml` file in the `src` folder of the repo. They select to build service images and include the latest tag. When the action changes to **Build service images**, the name of the Azure DevOps task changes to **Build services automatically**.

    ![Azure DevOps](./media/contoso-migration-rebuild/vsts9.png)
    _Figure 20: The specifics of the task._

11. Now, they configure the second Docker task (to push). They select the subscription and the container registry (`smarthotelacreus2`).

    ![Azure DevOps](./media/contoso-migration-rebuild/vsts10.png)
    _Figure 21: Configuring the second Docker task._

12. They enter the file to the `docker-compose.yaml` file and select **Push service images**, including the latest tag. When the action changes to **Push service images**, the name of the Azure DevOps task changes to **Push services automatically**.

    ![Azure DevOps](./media/contoso-migration-rebuild/vsts11.png)
    _Figure 22: Changing the Azure DevOps task name._

13. With the Azure DevOps tasks configured, Contoso saves the build pipeline and starts the build process.

    ![Azure DevOps](./media/contoso-migration-rebuild/vsts12.png)
    _Figure 23: Starting the build process._

14. They select the build job to check progress.

    ![Azure DevOps](./media/contoso-migration-rebuild/vsts13.png)
    _Figure 24: Checking the progress._

15. After the build finishes, the container registry shows the new repos, which are populated with the containers used by the microservices.

    ![Azure DevOps](./media/contoso-migration-rebuild/vsts14.png)
    _Figure 25: Viewing new repos after the build finishes._

### Deploy the back-end infrastructure

With the AKS cluster created and the Docker images built, Contoso admins now deploy the rest of the infrastructure that will be used by back-end microservices. Instructions in the section use the [SmartHotel360-Backend](https://github.com/Microsoft/SmartHotel360-Backend) repo. In the `/deploy/k8s/arm` folder, there's a single script to create all items.

They deploy as follows:

1. They open a developer command prompt, and use the command `az login` for the Azure subscription.

2. They use the `deploy.cmd` file to deploy the Azure resources in the `ContosoRG` resource group and `East US 2` region by typing the following command:

    `.\deploy.cmd azuredeploy ContosoRG -c eastus2`

    ![Deploy the back end](./media/contoso-migration-rebuild/backend1.png)
    _Figure 26: Deploying the back end._

3. In the Azure portal, they capture the connection string for each database to be used later.

    ![Deploy back-end](./media/contoso-migration-rebuild/backend2.png)
    _Figure 27: Capturing the connection string for each database._

### Create the back-end release pipeline

Now, Contoso admins do the following:

- Deploy the NGINX ingress controller to allow inbound traffic to the services.
- Deploy the microservices to the AKS cluster.
- As a first step they update the connection strings to the microservices using Azure DevOps. They then configure a new Azure DevOps release pipeline to deploy the microservices.
- The instructions in this section use the [SmartHotel360-Backend](https://github.com/Microsoft/SmartHotel360-Backend) repo.
- Some of the configuration settings (for example Active Directory B2C) aren't covered in this article. For more information about these settings, review the repo above.

They create the pipeline:

1. Using Visual Studio they update the `/deploy/k8s/config_local.yml` file with the database connection information they noted earlier.

    ![Database connections](./media/contoso-migration-rebuild/back-pipe1.png)
    _Figure 28: Database connections._

2. They open Azure DevOps and, in the SmartHotel360 project, they select **+ New Pipeline** in **Releases**.

    ![New pipeline](./media/contoso-migration-rebuild/back-pipe2.png)
    _Figure 29: The new pipeline.

3. They select **Empty Job** to start the pipeline without a template.
4. They provide the stage and pipeline names.

      ![Stage name](./media/contoso-migration-rebuild/back-pipe4.png)
        _Figure 30: The stage name._

      ![Pipeline name](./media/contoso-migration-rebuild/back-pipe5.png)
        _Figure 31: The pipeline name._

5. They add an artifact.

     ![Add artifact](./media/contoso-migration-rebuild/back-pipe6.png)
    _Figure 32: Adding an artifact._

6. They select **Git** as the source type and specify the project, source, and master branch for the SmartHotel360 application.

    ![Artifact settings](./media/contoso-migration-rebuild/back-pipe7.png)
    _Figure 33: The artifact settings._

7. They select the task link.

    ![Task link](./media/contoso-migration-rebuild/back-pipe8.png)
    _Figure 34: The task link._

8. They add a new Azure PowerShell task so that they can run a PowerShell script in an Azure environment.

    ![PowerShell in Azure](./media/contoso-migration-rebuild/back-pipe9.png)
    _Figure 35: Adding a new PowerShell task._

9. They select the Azure subscription for the task and select the `deploy.ps1` script from the Git repo.

    ![Run script](./media/contoso-migration-rebuild/back-pipe10.png)
    _Figure 36: Running the script._

10. They add arguments to the script. The script will delete all cluster content (except **ingress** and **ingress controller**), and deploy the microservices.

    ![Script arguments](./media/contoso-migration-rebuild/back-pipe11.png)
    _Figure 37: Adding arguments to the script._

11. They set the preferred Azure PowerShell version to the latest, and save the pipeline.

12. They move back to the **Release** page and manually create a new release.

    ![New release](./media/contoso-migration-rebuild/back-pipe12.png)
    _Figure 38: Manually creating a new release._

13. They select the release after creating it, and in **Actions**, they select **Deploy**.

      ![Deploy release](./media/contoso-migration-rebuild/back-pipe13.png)
    _Figure 39: Deploying a release._

14. When the deployment is complete, they run the following command to check the status of services, using the Azure Cloud Shell: `kubectl get services`.

## Step 3: Provision front-end services

Contoso admins need to deploy the infrastructure that will be used by the front-end applications. They create:

- A Blob storage container to store the pet images
- A Azure Cosmos DB database to store documents containing pet information
- The Computer Vision API for the website.

Instructions for this section use the [SmartHotel360-website](https://github.com/microsoft/smartHotel360-website) repo.

### Create Blob storage containers

1. In the Azure portal, they open the storage account that was created, then select **Blobs**.
2. They create a new container named `Pets` with the public-access level set for the container. Users will upload their pet photos to this container.

    ![Storage blob](./media/contoso-migration-rebuild/blob1.png)
    _Figure 40: Creating a new container._

3. They create a second new container named `settings`. A file with all the front-end app settings will be placed in this container.

    ![Storage blob](./media/contoso-migration-rebuild/blob2.png)
    _Figure 41: Creating a second container._

4. They capture the access details for the storage account in a text file for future reference.

    ![Storage blob](./media/contoso-migration-rebuild/blob2.png)
    _Figure 42: A text file capturing access details._

### Provision an Azure Cosmos DB database

Contoso admins provision an Azure Cosmos DB database to be used for pet information.

1. They create an **Azure Cosmos DB** database via the Azure Marketplace.

    ![Azure Cosmos DB](./media/contoso-migration-rebuild/cosmos1.png)
    _Figure 43: Creating an Azure Cosmos DB database._

2. They specify a name (`contososmarthotel`), select the SQL API, and place it in the production resource group `ContosoRG` in the main region (`East US 2`).

    ![Azure Cosmos DB](./media/contoso-migration-rebuild/cosmos2.png)
    _Figure 44: Naming an Azure Cosmos DB database._

3. They add a new collection to the database, with default capacity and throughput.

    ![Azure Cosmos DB](./media/contoso-migration-rebuild/cosmos3.png)
    _Figure 45: Adding a new collection to the database._

4. They note the connection information for the database for future reference.

    ![Azure Cosmos DB](./media/contoso-migration-rebuild/cosmos4.png)
    _Figure 46: The connection information for the database._

### Provision the Computer Vision API

Contoso admins provision the Computer Vision API. The API will be called by the function, to evaluate pictures uploaded by users.

1. They create a **Computer Vision** instance in the Azure Marketplace.

     ![Computer Vision](./media/contoso-migration-rebuild/vision1.png)
    _Figure 47: A new instance in the Azure Marketplace._

2. They provision the API (`smarthotelpets`) in the production resource group `ContosoRG`, in the main region (`East US 2`).

    ![Computer Vision](./media/contoso-migration-rebuild/vision2.png)
    _Figure 48: Provisioning an API in a production resource group._

3. They save the connection settings for the API to a text file for later reference.

     ![Computer Vision](./media/contoso-migration-rebuild/vision3.png)
    _Figure 49: Saving an API's connection's settings._

### Provision the Azure web app

Contoso admins provision the web app using the Azure portal.

1. They select **Web App** in the portal.

    ![Web app](./media/contoso-migration-rebuild/web-app1.png)
    _Figure 50: Selecting the web app._

2. They provide a web app name (`smarthotelcontoso`), run it on Windows, and place it in the production resource group `ContosoRG`. They create a new Application Insights instance for application monitoring.

    ![Web app name](./media/contoso-migration-rebuild/web-app2.png)
    _Figure 51: The web app name._

3. After they're done, they browse to the address of the application to check it's been created successfully.

4. In the Azure portal, they create a staging slot for the code. The pipeline will deploy to this slot. This ensures that code isn't put into production until admins perform a release.

    ![Web app staging slot](./media/contoso-migration-rebuild/web-app3.png)
    _Figure 52: A web app staging slot._

### Provision the function app

In the Azure portal, Contoso admins provision the function app.

1. They select **Function App**.

    ![Create a function app](./media/contoso-migration-rebuild/function-app1.png)
    _Figure 53: Selecting the function app._

2. They provide a name for the function app (`smarthotelpetchecker`). They place the function app in the production resource group (`ContosoRG`). They set the hosting place to **Consumption Plan**, and place the function app in the `East US 2` region. A new storage account is created along with an Application Insights instance for monitoring.

    ![Function app settings](./media/contoso-migration-rebuild/function-app2.png)
    _Figure 54: Function app settings._

3. After the function app is deployed, they browse to its address to verify it's been created successfully.

## Step 4: Set up the front-end pipeline

Contoso admins create two different projects for the front-end site.

1. In Azure DevOps, they create a project `SmartHotelFrontend`.

    ![Front-end project](./media/contoso-migration-rebuild/function-app1.png)
    _Figure 55: A front-end project._

2. They import the [SmartHotel360 front end](https://github.com/Microsoft/SmartHotel360-Website) Git repository into the new project.

3. For the function app, they create another Azure DevOps project (`SmartHotelPetChecker`) and import the [PetChecker](https://github.com/sonahander/SmartHotel360-PetCheckerFunction) Git repository into this project.

### Configure the web app

Now Contoso admins configure the web app to use Contoso resources.

1. They connect to the Azure DevOps project and clone the repository locally to the development machine.
2. In Visual Studio, they open the folder to show all the files in the repo.

    ![Repo files](./media/contoso-migration-rebuild/configure-webapp1.png)
    _Figure 56: Viewing all files in the repo._

3. They update the configuration changes as required.

    - When the web app starts up, it looks for the `SettingsUrl` app setting.
    - This variable must contain a URL pointing to a configuration file.
    - By default, the setting used is a public endpoint.

4. They update the `/config-sample.json/sample.json` file.

    - This is the configuration file for the web when using the public endpoint.
    - They edit the `urls` and `pets_config` sections with the values for the AKS API endpoints, storage accounts, and Azure Cosmos DB database.
    - The URLs should match the DNS name of the new web app that Contoso will create.
    - For Contoso, this is `smarthotelcontoso.eastus2.cloudapp.azure.com`.

    ![Json settings](./media/contoso-migration-rebuild/configure-webapp2.png)
    _Figure 57: .json settings._

5. After the file is updated, they rename it `smarthotelsettingsurl` and upload it to the Blob storage they created earlier.

    ![Rename and upload](./media/contoso-migration-rebuild/configure-webapp3.png)
    _Figure 58: Renaming and uploading the file._

6. They select the file to get the URL. The URL is used by the application when it pulls down the configuration files.

    ![App URL](./media/contoso-migration-rebuild/configure-webapp4.png)
    _Figure 59: The application URL._

7. In the `appsettings.Production.json` file, they update the `SettingsURL` to the URL of the new file.

    ![Update URL](./media/contoso-migration-rebuild/configure-webapp5.png)
    _Figure 60: Updating the URL to the new file._

### Deploy the website to Azure App Service

Contoso admins can now publish the website.

1. They open Azure DevOps, and in the `SmartHotelFrontend` project in **Builds and Releases**, they select **+ New Pipeline**.
2. They select **Azure DevOps Git** as a source.
3. They select the **ASP.NET Core** template.
4. They review the pipeline and check that **Publish Web Projects** and **Zip Published Projects** are selected.

    ![Pipeline settings](./media/contoso-migration-rebuild/vsts-publish-front2.png)
    _Figure 61: Pipeline settings._

5. In **Triggers**, they enable continuous integration and add the master branch. This ensures that the build pipeline starts each time the solution has new code committed to the master branch.

    ![Continuous integration](./media/contoso-migration-rebuild/vsts-publish-front3.png)
    _Figure 62: Continuous integration._

6. They select **Save & Queue** to start a build.
7. After the build completes, they configure a release pipeline using **Azure App Service Deployment**.
8. They provide a stage name **Staging**.

    ![Environment name](./media/contoso-migration-rebuild/vsts-publish-front4.png)
    _Figure 63: Naming the environment._

9. They add an artifact and select the build that they configured.

    ![Add artifact](./media/contoso-migration-rebuild/vsts-publish-front5.png)
    _Figure 64: Adding an artifact._

10. They select the lightning bolt icon on the artifact and enable continuous deployment.

    ![Continuous deployment](./media/contoso-migration-rebuild/vsts-publish-front6.png)
    _Figure 65: Enabling continuous deployment._

11. In **Environment**, they select **1 job, 1 task** under **Staging**.
12. After selecting the subscription and web app name, they open the **Deploy Azure App Service** task. The deployment is configured to use the **staging** deployment slot. This automatically builds code for review and approval in this slot.

     ![Slot](./media/contoso-migration-rebuild/vsts-publish-front7.png)
    _Figure 66: Using a slot._

13. In the **Pipeline**, they add a new stage.

    ![New environment](./media/contoso-migration-rebuild/vsts-publish-front8.png)
    _Figure 67: Adding a new stage._

14. They select **Azure App Service deployment with slot**, and name the environment **Prod**.
15. They select **1 job, 2 tasks**, then select the subscription, the app service name, and the **staging** slot.

    ![Environment name](./media/contoso-migration-rebuild/vsts-publish-front10.png)
    _Figure 68: An environment name._

16. They remove the **Deploy Azure App Service to Slot** from the pipeline. It was placed there by the previous steps.

    ![Remove from pipeline](./media/contoso-migration-rebuild/vsts-publish-front11.png)
    _Figure 69: Removing a slot from the pipeline._

17. They save the pipeline. On the pipeline, they select **Post-deployment conditions**.

    ![Post-deployment](./media/contoso-migration-rebuild/vsts-publish-front12.png)
    _Figure 70: Post-deployment conditions._

18. They enable **Post-deployment approvals** and add a dev lead as the approver.

    ![Post-deployment approval](./media/contoso-migration-rebuild/vsts-publish-front13.png)
    _Figure 71: Adding an approver._

19. In the build pipeline, they manually kick off a build. This triggers the new release pipeline, which deploys the site to the staging slot. For Contoso, the URL for the slot is `https://smarthotelcontoso-staging.azurewebsites.net/`.

20. After the build finishes and the release deploys to the slot, Azure DevOps emails the dev lead for approval.

21. The dev lead selects **View approval** and can approve or reject the request in the Azure DevOps portal.

    ![Approval mail](./media/contoso-migration-rebuild/vsts-publish-front14.png)
    _Figure 72: A dev lead approving a request._

22. The lead makes a comment and approves. This starts swapping the **staging** and **prod** slots and moves the build into production.

    ![Approve and swap](./media/contoso-migration-rebuild/vsts-publish-front15.png)
    _Figure 73: Moving the build into production._

23. The pipeline completes the swap.

    ![Complete swap](./media/contoso-migration-rebuild/vsts-publish-front16.png)
    _Figure 74: Completing the swap._

24. The team checks the **prod** slot to verify that the web app is in production at `https://smarthotelcontoso.azurewebsites.net/`.

### Deploy the PetChecker function app

Contoso admins deploy the application:

1. They clone the repository locally to the development machine by connecting to the Azure DevOps project.
2. In Visual Studio, they open the folder to show all the files in the repo.
3. They open the `src/PetCheckerFunction/local.settings.json` file and add the app settings for storage, the Azure Cosmos DB database, and the Computer Vision API.

    ![Deploy the function](./media/contoso-migration-rebuild/function5.png)
    _Figure 75: Deploying the function._

4. They commit the code and sync it back to Azure DevOps, pushing their changes.
5. They add a new build pipeline, then select **Azure DevOps Git** for the source.
6. They select the **ASP.NET Core (.NET Framework)** template.
7. They accept the defaults for the template.
8. In **Triggers**, they select **Enable continuous integration**, then select **Save & Queue** to start a build.
9. After the build succeeds, they build a release pipeline, adding **Azure App Service deployment with slot**.
10. They name the environment **Prod**, then select the subscription. They set the **App type** to **Function App** and the app service name as `smarthotelpetchecker`.

    ![Function app](./media/contoso-migration-rebuild/petchecker2.png)
    _Figure 76: The function app._

11. They add an artifact **Build**.

    ![Add an artifact](./media/contoso-migration-rebuild/petchecker3.png)
    _Figure 77: Add an artifact._

12. They enable **Continuous deployment trigger**, then select **Save**.
13. They select **Queue new build** to run the full CI/CD pipeline.
14. After the function is deployed, it appears in the Azure portal with the status **Running** .

    ![Update the function's status](./media/contoso-migration-rebuild/function6.png)
    _Figure 78: Update the function's status._

15. They browse to the Pet Checker application to verify that it's working properly, at `http://smarthotel360public.azurewebsites.net/pets`.

16. They select the avatar to upload a picture.

    ![Assign a picture to an avatar](./media/contoso-migration-rebuild/function7.png)
    _Figure 79: Assign a picture to an avatar._

17. The first photo they want to check is of a small dog.

    ![Check a photo](./media/contoso-migration-rebuild/function8.png)
    _Figure 80: Check a photo._

18. The application returns a message of acceptance.

    ![An acceptance message](./media/contoso-migration-rebuild/function9.png)
    _Figure 81: An acceptance message._

## Review the deployment

With the migrated resources in Azure, Contoso now needs to fully operationalize and secure the new infrastructure.

### Security

- Contoso needs to ensure that the new databases are secure. [Learn more](https://docs.microsoft.com/azure/sql-database/sql-database-security-overview).
- The application must be updated to use SSL with certificates. The container instance should be redeployed to answer on 443.
- Contoso should consider using Key Vault to protect secrets for their Service Fabric applications. [Learn more](https://docs.microsoft.com/azure/service-fabric/service-fabric-application-secret-management).

### Backups and disaster recovery

- Contoso needs to review [backup requirements for the Azure SQL Database](https://docs.microsoft.com/azure/sql-database/sql-database-automated-backups).
- Contoso should consider implementing [SQL failover groups to provide regional failover for the database](https://docs.microsoft.com/azure/sql-database/sql-database-auto-failover-group).
- Contoso can use [geo-replication for the Azure Container Registry Premium SKU](https://docs.microsoft.com/azure/container-registry/container-registry-geo-replication).
- Azure Cosmos DB is backed up automatically. Contoso can [learn more](https://docs.microsoft.com/azure/cosmos-db/online-backup-and-restore) about this process.

### Licensing and cost optimization

- After all resources are deployed, Contoso should assign Azure tags based on their [infrastructure planning](./contoso-migration-infrastructure.md#set-up-tagging).
- All licensing is built into the cost of the PaaS services that Contoso is consuming. This will be deducted from the EA.
- Contoso will enable [Azure Cost Management and Billing](https://docs.microsoft.com/azure/cost-management-billing/cost-management-billing-overview) to help monitor and manage the Azure resources.

## Conclusion

In this article, Contoso rebuilds the SmartHotel360 application in Azure. The on-premises application front-end VM is rebuilt to Azure App Service web apps. The application back end is built using microservices deployed to containers managed by AKS. Contoso enhanced functionality with a pet photo application.

## Suggested skills

Microsoft Learn is a new approach to learning. Readiness for the new skills and responsibilities that come with cloud adoption doesn't come easily. Microsoft Learn provides a more rewarding approach to hands-on learning that helps you achieve your goals faster. Earn points and levels, and achieve more!

Here are several examples of tailored learning paths on Microsoft Learn that align with the Contoso SmartHotel360 application in Azure.

<!--docsTest:ignore "Azure Cognitive Vision Services" -->

- **[Deploy a website to Azure with Azure App Service](https://docs.microsoft.com/learn/paths/deploy-a-website-with-azure-app-service):** Web apps in Azure allow you to publish and manage your website easily without having to work with the underlying servers, storage, or network assets. Instead, you can focus on your website features and rely on the robust Azure platform to provide secure access to your site.

- **[Process and classify images with the Azure cognitive vision services](https://docs.microsoft.com/learn/paths/classify-images-with-vision-services):** Azure Cognitive Services offers prebuilt functionality to enable computer vision functionality in your applications. Learn how to use the cognitive vision services to detect faces, tag and classify images, and identify objects.
