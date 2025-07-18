# Runbook: Deploying and Migrating to a Zone-Redundant Azure Container Apps (ACA) Environment



## 1\. Purpose

This runbook provides a detailed guide for architects and engineers to:

  * Create a **zone-redundant Azure Container Apps (ACA) environment** with supporting infrastructure.
  * Migrate existing non-zone-redundant ACA environments including Azure Container Registry (ACR) migration.
  * Automate provisioning and deployments via Infrastructure as Code (IaC) using **Bicep**.
  * Outline application deployment and migration validation steps.



## 2\. Scope

This guide applies to applications hosted on **Azure Container Apps (ACA)** with zone redundancy enabled. It covers:

  * ACA environment creation (zone redundant)
  * Azure Blob Storage (Zone-Redundant Storage - ZRS)
  * Azure Container Registry (Premium SKU with zone redundancy)
  * Migration strategies for existing environments
  * Application deployment automation with IaC (Bicep)
  * Post-deployment validation



## 3\. Assumptions and Prerequisites

Before using this runbook, the following prerequisites must be met:

  * The target Azure region supports **Availability Zones** (e.g., East US, West Europe).
  * **A dedicated subnet within your existing virtual network is available for ACA deployment.**
> [!IMPORTANT]  
> In enterprise environments, network subnets are typically managed by a separate networking team. Please ensure this subnet has been **requested and provided by your networking team** before proceeding.
  * Sufficient Azure **Role-Based Access Control (RBAC)** permissions are available for resource provisioning.
  * Infrastructure as Code templates (**Bicep**) are maintained in a version-controlled repository (e.g., GitHub, Azure Repos).
  * CI/CD pipelines are configured (using Azure DevOps, GitHub Actions, etc.).
  * A jump host or deployment agent is available inside the virtual network if the environment is isolated.
  * The customer team is responsible for dependent services, external integrations, and data migration.
  * Applications are containerized and packaged using Docker or similar tooling.



## 4\. Architecture Overview

```
Internet → [Azure Front Door / Application Gateway (Zone Redundant)] → Your Virtual Network (Spoke)
                                                                           ↓
                                                                           ACA Environment (Zone Redundant)
                                                                               ↘ Azure Blob Storage (ZRS)
                                                                               ↘ Dependent Services (if zone-redundant, e.g., databases, caches)
                                                                               ↘ Azure Container Registry (Premium, Zone Redundant)
```

### 4.1. Key Components

| Component                          | Purpose                                                    |
| :--------------------------------- | :--------------------------------------------------------- |
| **Azure Container Apps (ACA)** | Hosts microservices and containerized workloads.           |
| **Azure Virtual Network (VNet)** | Provides isolation and connectivity for the ACA environment. |
| **Subnets** | Segments within VNet for ACA, ACR, Storage.                |
| **Azure Container Registry (ACR)** | Stores container images; must be Premium SKU for ZRS.      |
| **Azure Blob Storage (ZRS)** | Stores artifacts, logs, configs with Zone Redundant Storage. |
| **Azure Log Analytics** | Monitoring and logging.                                    |
| **Azure Key Vault** | Secrets and certificates management.                       |
| **Jump Host** | VM used to perform deployments inside isolated VNet.       |

### 4.2. Zonal Design Considerations

  * ACA supports zone redundancy using the `zoneRedundant: true` configuration.
  * **ACR Premium** provides automatic replication across zones.
  * Blob Storage must be deployed with `sku: ZRS` to ensure redundancy across availability zones.



## 5\. Provisioning the Infrastructure with Bicep

Infrastructure should be provisioned using Infrastructure as Code (IaC) for consistency and automation.

### 5.1. Zone-Redundant Azure Container Apps Environment

| **Audience** | **Expected Duration** | **Expected Outcomes** |
| :---------------------------------------------- | :-------------------- | :--------------------------------------------------------------- |
| Customer: Application Architects, DevOps        | 1 hour                | Zone-redundant ACA environment deployed and configured           |
| Microsoft Cloud Accelerator Factory: Lead Architect |                       | Validate configuration aligns with zonal redundancy requirements |

```bicep
resource acaEnv 'Microsoft.App/managedEnvironments@2023-05-01' = {
  name: 'aca-zonal-env'
  location: location // Parameterized location, e.g., 'eastus'
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logAnalyticsWorkspace.properties.customerId
        sharedKey: logAnalyticsWorkspace.listKeys().primarySharedKey
      }
    }
    zoneRedundant: true
    vnetConfiguration: {
      infrastructureSubnetId: infrastructureSubnetId // Parameter for your existing subnet ID
    }
  }
}
```

> [!NOTE]  
> `zoneRedundant: true` must be set **at creation**. This setting is immutable. The `infrastructureSubnetId` should refer to the dedicated subnet provided by your networking team.

### 5.2. Azure Container Registry (ACR) with Zone Redundancy

| **Audience** | **Expected Duration** | **Expected Outcomes** |
| :---------------------------------------------- | :-------------------- | :---------------------------------------------------- |
| Customer: DevOps, Application Teams             | 1 hour                | Premium SKU ACR deployed with zone redundancy enabled |
| Microsoft Cloud Accelerator Factory: Lead Architect |                       | Validate ACR zone redundancy configuration            |

```bicep
resource acr 'Microsoft.ContainerRegistry/registries@2023-01-01-preview' = {
  name: 'myacr' // Your ACR name
  location: location
  sku: {
    name: 'Premium'
  }
  properties: {
    zoneRedundancy: 'Enabled'
    adminUserEnabled: true
  }
}
```

### 5.3. Azure Blob Storage (ZRS)

| **Audience** | **Expected Duration** | **Expected Outcomes** |
| :---------------------------------------------- | :-------------------- | :----------------------------------------------- |
| Customer: Storage Admins, DevOps                | 1 hour                | Storage accounts created with ZRS SKU            |
| Microsoft Cloud Accelerator Factory: Lead Architect |                       | Ensure compliance with zone redundancy standards |

```bicep
resource storage 'Microsoft.Storage/storageAccounts@2022-09-01' = {
  name: 'mystorageacct' // Your storage account name
  location: location
  sku: {
    name: 'Standard_ZRS'
  }
  kind: 'StorageV2'
  properties: {
    allowBlobPublicAccess: false
    minimumTlsVersion: 'TLS1_2'
  }
}
```



## 6\. Migration Strategy for Existing Environments

Migrating existing non-zone-redundant Azure Container Apps (ACA) environments to a zone-redundant setup requires careful planning and execution. This section outlines the key phases and steps involved.

### 6.1. Inventory and Assessment

This crucial initial phase involves understanding your current environment to inform the migration plan.

| **Audience** | **Expected Duration** | **Expected Outcomes** |
| :---------------------------------------------- | :-------------------- | :------------------------------------------------------------------------------- |
| Customer: Application Architects, DevOps        | 2-4 hours             | Comprehensive list of existing ACA apps and their configurations; identified dependencies |
| Microsoft Cloud Accelerator Factory: Lead Architect |                       | Clear understanding of the current environment and assistance with configuration export |

**Steps:**

1.  **List Existing ACA Applications:** Identify all Azure Container Apps deployed in your current, non-zone-redundant environment.
2.  **Document Configurations:** For each application, extract and document its full configuration, including:
      * Ingress settings (internal/external, target port, traffic weighting).
      * Environment variables.
      * Secrets (references to Azure Key Vault or directly managed secrets).
      * Custom domains and SSL certificates.
      * Scale rules (min/max replicas, CPU/memory scaling).
      * Probes (liveness, readiness, startup).
      * Container image references.
      * Linked storage (e.g., Azure File shares).
3.  **Identify Dependencies:** Map out all external and internal dependencies for each application:
      * Other ACA applications.
      * Azure services (e.g., databases, caches like Azure Cache for Redis, message queues like Azure Service Bus, Azure Event Hubs).
      * External APIs or services.
      * Azure Key Vault secrets or certificates used.
4.  **Export Current Configurations (for reference):** Use Azure CLI or ARM template export features to capture the current state of your ACA environments and individual applications. This provides a baseline and helps in recreating settings.
    ```bash
    # Export an ACA environment ARM template
    az deployment group export --resource-group <your-resource-group> --name <your-aca-environment-name> > aca-env-old.json

    # Export a specific Container App ARM template
    az group export --resource-group <your-resource-group> --query "[?type=='Microsoft.App/containerApps' && name=='<your-container-app-name>']" --output json > aca-app-old.json
    ```
5.  **Review Network Configuration:** Understand how your current ACA environment is connected to other resources. This will inform the networking requirements for the new zone-redundant environment.

### 6.2. New Zone-Redundant Infrastructure Provisioning (ACA Environment, ACR, Storage)

Since existing ACA environments **cannot be converted** to zone-redundant, a new environment must be provisioned alongside its supporting services.

| **Audience** | **Expected Duration** | **Expected Outcomes** |
| :---------------------------------------------- | :-------------------- | :--------------------------------------------------------------- |
| Customer: Application Architects, DevOps        | 1-2 days (per environment) | New zone-redundant ACA environment, ACR, and ZRS storage provisioned and ready |
| Microsoft Cloud Accelerator Factory: Lead Architect |                       | Guide customer on recreating environments using IaC, validate zonal configuration |

**Steps:**

1.  **Request Subnet:** Ensure a **dedicated subnet** within your existing virtual network has been provided by your networking team (as per prerequisites).
2.  **Provision Zone-Redundant ACA Environment:** Deploy the new ACA environment using **Bicep** as detailed in **Section 5.1**. Ensure `zoneRedundant: true` is explicitly set.
3.  **Provision Zone-Redundant Azure Container Registry (ACR):** Deploy a new ACR with the `Premium` SKU and `zoneRedundancy: 'Enabled'` as described in **Section 5.2**. This will be the target for your container images.
4.  **Provision Zone-Redundant Azure Blob Storage (ZRS):** Deploy any necessary Azure Blob Storage accounts with `Standard_ZRS` SKU as shown in **Section 5.3**. Update application configurations to point to these new storage accounts where applicable.
5.  **Recreate Supporting Services Configurations:** If your applications rely on Azure Key Vault or Log Analytics, ensure the new ACA environment is correctly configured to integrate with these services.

### 6.3. Container Image Migration

Transfer your necessary container images from the old ACR to the newly provisioned zone-redundant Premium ACR.

| **Audience** | **Expected Duration** | **Expected Outcomes** |
| :---------------------------------------------- | :-------------------- | :------------------------------------------------------- |
| Customer: DevOps, Application Teams             | Varies (image count)  | All necessary container images available in new zone-redundant ACR |
| Microsoft Cloud Accelerator Factory: Lead Architect |                       | Assist with image transfer and pipeline setup guidance   |

**Steps:**

1.  **Login to Registries:** Authenticate with both your old (source) and new (target) Azure Container Registries.

    ```bash
    az acr login --name <old-acr-name>
    az acr login --name <new-acr-name>
    ```

2.  **Migrate Images (Choose one method):**

      * **Method A: Manual Docker CLI (for a few images):**

        ```bash
        docker pull <old-acr-name>.azurecr.io/<image-name>:<tag>
        docker tag <old-acr-name>.azurecr.io/<image-name>:<tag> <new-acr-name>.azurecr.io/<image-name>:<tag>
        docker push <new-acr-name>.azurecr.io/<image-name>:<tag>
        ```

        Repeat for all required images and tags.

      * **Method B: Use the `acr-transfer` tool (recommended for bulk migration):**
        This tool facilitates efficient transfer of multiple images and repositories between registries.

        ```bash
        git clone https://github.com/Azure/acr-transfer
        cd acr-transfer
        python3 transfer.py \
          --source <old-acr-name>.azurecr.io \
          --destination <new-acr-name>.azurecr.io \
          --repository <repository-name> # Or --all for all repositories
        ```

3.  **Update CI/CD Pipelines:** Crucially, modify all your CI/CD pipelines (e.g., Azure DevOps, GitHub Actions) to:

      * Reference the **new zone-redundant ACR URL** for pulling base images and pushing built images.
      * Ensure any service principals or managed identities used by pipelines have permissions to the new ACR.

### 6.4. Application Configuration Replication & Deployment

Recreate and deploy your applications in the new zone-redundant ACA environment, adapting to any changes in dependent service endpoints or configurations.

| **Audience** | **Expected Duration** | **Expected Outcomes** |
| :---------------------------------------------- | :-------------------- | :--------------------------------------------------------------- |
| Customer: Developers, DevOps                    | 1-2 hours (per app)   | Applications deployed successfully into zone-redundant ACA environment |
| Microsoft Cloud Accelerator Factory: Lead Architect |                       | Review deployment manifests and monitor deployment              |

**Steps:**

1.  **Adapt Application Configurations:** Based on your assessment (Section 6.1), update application configurations (e.g., environment variables, connection strings) to point to any new zone-redundant dependent services (e.g., new database instances, new cache instances) or the new ACR.
2.  **Recreate Secrets:** If secrets were directly managed in the old ACA environment, recreate them in the new environment. If using Azure Key Vault, ensure your new ACA environment has the correct managed identity or service principal permissions to access the existing Key Vault.
3.  **Deploy Applications:** Deploy each container application to the new zone-redundant ACA environment using your CI/CD pipelines and the updated **Bicep** manifests (as outlined in **Section 7**). Ensure `minReplicas` is set to at least `3` for zonal distribution.

### 6.5. Dependent Services Migration (Customer Responsibility)

Migration of dependent services is a critical step, but is outside the direct scope of the **Microsoft Cloud Accelerator Factory**.

| **Audience** | **Expected Duration** | **Expected Outcomes** |
| :------------------------------------------ | :-------------------- | :-------------------------------------------------------- |
| Customer: Application Teams, DBAs           | Varies                | Dependent services migrated or deployed by customer teams |
| Microsoft Cloud Accelerator Factory: Lead Architect |                       | Coordinate migration alignment guidance                   |

> [!NOTE]  
>
> - Customer teams are solely responsible for planning and executing the migration or deployment of all dependent services (e.g., databases, caches, message queues, specialized storage, external APIs).
> - Prioritize migrating dependent services that offer **zone-redundant or zone-aware** deployment options to align with the new ACA environment's high-availability design.
> - Ensure network connectivity (VNet peering, firewall rules) is configured to allow communication between the new ACA environment and these migrated dependent services.

### 6.6. Data / State Migration (Customer Responsibility)

Data migration is a critical path for many applications and remains the customer's responsibility.

| **Audience** | **Expected Duration** | **Expected Outcomes** |
| :------------------------------------------ | :-------------------- | :-------------------------------------------- |
| Customer: Database Teams, Application Teams | Varies                | Data/state migrated securely and successfully |
| Microsoft Cloud Accelerator Factory: Lead Architect |                       | Validate migration readiness and provide guidance on tools |

**Recommended Tools and Considerations:**

  * **Databases:** Utilize native database backup/restore, replication technologies, or Azure Data Migration Service (DMS).
  * **Blob Storage:** Use AzCopy, Azure Data Factory, or Storage Explorer for transferring data between storage accounts.
  * **Persistent Volumes (if used):** Leverage tools like AzCopy for file shares, or specialized solutions like CloudCasa for Kubernetes-native persistent volumes (if applicable for non-managed ACA persistent storage).
  * **Connectivity:** Ensure robust network connectivity and appropriate firewall rules are in place to facilitate secure and efficient data transfer between source and target environments.

### 6.7. Traffic Routing Strategy (Cutover)

Plan and execute the cutover of live traffic to the new zone-redundant environment.

| **Audience** | **Expected Duration** | **Expected Outcomes** |
| :-------------------------------------------- | :-------------------- | :--------------------------------------------------------------- |
| Customer: Network Architects, DevOps, Application Owners | 2-4 hours             | Controlled shift of live traffic to the new zone-redundant environment |
| Microsoft Cloud Accelerator Factory: Lead Architect |                       | Provide best practices for external traffic management and blue-green deployments |

**Steps:**

1.  **Blue-Green Deployment (Recommended):**
      * Deploy the new zone-redundant environment alongside the old non-zonal environment.
      * Use Azure Front Door, Azure Application Gateway, or DNS changes to gradually shift traffic from the old environment to the new one. This allows for controlled testing and quick rollback if issues arise.
2.  **DNS Updates:** Update DNS records (CNAME or A records) to point to the ingress endpoint of the new zone-redundant ACA environment. Manage TTLs appropriately for controlled propagation.
3.  **Monitor Cutover:** Closely monitor application performance, errors, and logs during and immediately after the traffic cutover.



## 7\. Application Deployment

| **Audience** | **Expected Duration** | **Expected Outcomes** |
| :---------------------------------------------- | :-------------------- | :--------------------------------------------------------------------- |
| Customer: Developers, DevOps                    | 1-2 hours (per app)   | Applications deployed successfully into zone-redundant ACA environment |
| Microsoft Cloud Accelerator Factory: Lead Architect |                       | Review deployment manifests and monitor deployment                     |

  * Ensure CI/CD pipelines (Azure DevOps, GitHub Actions) are configured to:
      * Build and push images to the new zone-redundant ACR.
      * Deploy ACA applications using **Bicep** or Azure CLI.
      * Retrieve secrets from Azure Key Vault.
  * When deploying container apps, configure `minReplicas` to at least `3` to ensure workload distribution across availability zones.

### 7.1. Bicep Example for Container App Deployment

```bicep
resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: 'myapp' // Your application name
  location: location
  properties: {
    environmentId: acaEnv.id // Reference to your zone-redundant ACA environment
    configuration: {
      ingress: {
        external: true
        targetPort: 80 # Or your application's port
      }
    }
    template: {
      containers: [
        {
          name: 'my-app-container'
          image: 'myacr.azurecr.io/myapp:$(Build.BuildId)' // Reference to new ACR and build ID
        }
      ]
      scale: {
        minReplicas: 3 // Essential for zonal distribution
        maxReplicas: 10
      }
    }
  }
}
```



## 8\. Post-Migration Validation

### 8.1. Smoke Testing

| **Audience** | **Expected Duration** | **Expected Outcomes** |
| :-------------------------------------------- | :-------------------- | :-------------------------------------------------------------------- |
| Customer: QA Teams, Application Owners        | Varies (app complexity) | Core application functionalities validated in the new environment     |
| Microsoft Cloud Accelerator Factory: Lead Architect |                       | Confirm initial connectivity and basic configuration correctness      |

  * Verify ACA applications are accessible over intended ingress endpoints.
  * Test key application features and endpoints.
  * Validate configuration variables, secrets, and logging.

### 8.2. Monitoring Setup

  * Confirm logs are flowing to Azure Log Analytics.
  * Enable alert rules for ACA and ACR health metrics.

### 8.3. Backup and Failover Readiness

  * Confirm that backup procedures (if any) are valid post-migration for all dependent services.

  * Review zone-aware deployment correctness using:

    ```bash
    az containerapp show --name myapp --resource-group myrg --query properties.zoneRedundant
    ```



## 9\. Application Validation

| **Audience** | **Expected Duration** | **Expected Outcomes** |
| :------------------------------------- | :-------------------- | :-------------------------------------------------------------------- |
| Customer: QA Teams, Application Owners | Customer dependent    | Application validated and tested successfully                         |
| Microsoft Cloud Accelerator Factory: Project Manager |                       | Raise UAT requests for blockers requiring global Microsoft assistance |

  * Application testing and validation, including Functional Testing, End-to-End Testing, Regression Testing, and User Acceptance Testing (UAT), are **out of scope** for the Microsoft Cloud Accelerator Factory. This is the sole responsibility of the customer team.
  > [!IMPORTANT] 
  > If the customer is blocked or has challenges migrating their application at any step of the process that is outside the scope of the Microsoft Cloud Accelerator Factory, or if the Cloud Accelerator Factory team requires assistance, a UAT request must be raised by the local Microsoft field team to engage the Global Microsoft teams.



## 10\. Responsibilities and Handoff

| Task                                  | Microsoft Cloud Accelerator Factory Responsibility         | Customer Responsibility                                    |
| :------------------------------------ | :--------------------------------------------------------- | :--------------------------------------------------------- |
| Infrastructure provisioning (Bicep)   | Define architecture, author Bicep templates, deploy core resources | Approve architecture and regional setup, provide access, monitor billing |
| ACA and ACR environment setup         | Deploy ACA and Premium ACR with zone redundancy            | Review and validate environment configuration              |
| IaC pipeline setup (core infrastructure) | Set up initial CI/CD pipelines for infrastructure provisioning | Own, maintain, and execute CI/CD pipelines for application deployment |
| Application configuration migration   | Assist in exporting/recreating ACA configurations          | Responsible for actual application configuration and secret management in new environment |
| App deployment via CI/CD (guidance)   | Provide guidance and support for application deployment via CI/CD | Deploy and validate applications in the new environment    |
| Application validation and testing    | Provide validation checklist and initial smoke tests       | **Full functional, end-to-end, and UAT testing** |
| Data migration                        | Provide guidance on tools for data migration               | **Execute all data migration activities** |
| Dependent services migration          | Guidance on zonal options for dependent services           | **Plan and execute migration of all dependent services** |
| Production cutover decision           | Advise on readiness criteria                               | **Final decision on production cutover** |
| Knowledge Transfer & Handoff          | Provide documentation, walkthrough, clean IaC artifacts    | Own ongoing operations, monitoring, and future enhancements |



## 11\. Support and Escalation

If issues arise post-deployment:

  * Use **Azure Monitor** and **Log Analytics** for diagnostics.
  * Open a support request via Azure Portal if required.
  * Contact the local Microsoft field engineer for urgent escalation or UAT request initiation for issues beyond the scope of the Microsoft Cloud Accelerator Factory.



## 12\. References and Resources

  * [Azure Container Apps Documentation](https://learn.microsoft.com/en-us/azure/container-apps/)
  * [Zone Redundancy in Azure](https://learn.microsoft.com/en-us/azure/availability-zones/)
  * [ACR Transfer Tool GitHub Repository](https://github.com/Azure/acr-transfer)
  * [Bicep Documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)

