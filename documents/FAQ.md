# Lab FAQ

This FAQ covers the deployment and operational issues most commonly encountered when running the Agentic Applications for Unified Data Foundation lab with Microsoft Fabric and Azure AI Foundry.

## Before deploying

### What permissions does the deployment user need?

The deployment user needs permission to create Azure resources, create Microsoft Entra application registrations, and create Azure role assignments. `Owner` at subscription scope is the simplest lab setup. In a least-privilege environment, combine resource-creation permissions with `User Access Administrator` where role assignments are required.

Fabric also needs a tenant member who can administer the capacity and workspace. A B2B guest cannot be a Fabric capacity administrator.

### Which Fabric tenant settings must be enabled?

In the [Fabric admin portal](https://app.fabric.microsoft.com/admin-portal), enable these tenant settings for the deployment users:

- **Ontology (preview)**
- **Graph (preview)**
- **Copilot and Azure OpenAI Service**

Allow up to 15 minutes for tenant-setting changes to propagate before initializing the Fabric scenario.

### What local tools are required?

Use PowerShell 7+, Azure Developer CLI (`azd`), Azure CLI (`az`), Bicep, Python 3.9+, Docker Desktop, Git, and Microsoft ODBC Driver 17. See the [Deployment Guide](./DeploymentGuide.md) for supported environment options.

### How much Fabric capacity is required?

The lab requires an F2 or higher Fabric capacity. The capacity must be available in a region that supports the Fabric workloads used by the selected scenario.

### How do I choose a region?

Check all of the following before provisioning:

1. App Service plan quota and regional availability.
2. Azure AI Foundry availability and model quota for both the chat and embedding deployments.
3. Fabric capacity availability and required Fabric workloads.
4. Azure Container Registry, Azure AI Search, Storage, Cosmos DB, and monitoring availability.

The default region is not guaranteed to have sufficient quota in every subscription. The [Quota Check](./QuotaCheck.md) guide covers model quota; App Service quotas must also be checked separately.

### Can I deploy resources in more than one region?

Yes. This lab supports a split-region deployment when a single region cannot host every required service. Keep the application-facing resources together where possible, and place Fabric, Foundry, and Cosmos DB in a compatible region with sufficient capacity. Configure the locations through the documented `azd` environment parameters in [Customizing azd Parameters](./CustomizingAzdParameters.md).

### Azure reports zero App Service VM quota. What should I do?

This is a subscription quota constraint, not a template error. Check both the requested App Service SKU quota and the aggregate **Total Regional VMs** quota for the region. If either is zero, request quota through the Azure portal or choose another region with capacity. Increasing the App Service SKU does not bypass a zero aggregate regional VM quota.

### My deployment fails during a What-If or preview. Is the deployment itself broken?

Not necessarily. A large nested deployment can exceed the Azure Resource Manager What-If request-size limit even when normal incremental deployment succeeds. Validate Bicep compilation and parameter mappings, then use the regular incremental deployment path if the preview error specifically reports a request-size limit.

## Deploying and initializing the scenario

### Why does `azd up` ask for values that are already in my environment?

`azd` may prompt for missing or invalid environment values. Configure the environment first with `azd env set`, then rerun the command. Avoid placing array-valued Bicep parameters directly in an `azd` environment unless the template explicitly accepts a JSON string and parses it.

### The Fabric capacity deployment succeeds but scenario initialization fails. What should I check?

Confirm that:

- The capacity is active and has at least F2 capacity.
- The workspace is assigned to that capacity.
- The Fabric tenant settings are enabled and fully propagated.
- The identity running initialization is a tenant member with workspace and capacity administrative access.

### Why cannot my guest account administer the Fabric capacity?

Fabric capacity administration requires a tenant-member account. Use a member account in the tenant for the capacity administrator, then grant the guest access to the workspace and Azure resources as needed.

### Why does Azure AI Search return 403 during scenario initialization?

The initialization identity needs Azure AI Search data-plane permissions, not only management-plane permissions. Assign **Search Index Data Contributor** to write index documents; **Search Service Contributor** may also be required for service-level setup actions.

### Can I reuse an existing Foundry project or Log Analytics workspace?

Yes. Set the corresponding environment parameters before deployment. Follow [Reusing an Existing Azure AI Foundry Project](./re-use-foundry-project.md) and [Reusing an Existing Log Analytics Workspace](./re-use-log-analytics.md), and ensure the deployment identity has the required roles on the reused resource.

## Sign-in and web chat

### The application loads, but sending a message returns 405. What causes this?

The frontend reverse proxy is not configured for the API. Ensure the frontend App Service has `BACKEND_API_HOST` set to the API host name. The frontend container uses this setting to generate its `/api` proxy; without it, Nginx handles `POST /api/chat` as a static-site request.

### The application loads, but every message fails after enabling sign-in. What should I check?

Use the same-origin `/api` endpoint from the browser. Do not configure the browser to call the API App Service directly, because that bypasses the frontend proxy and its token forwarding. Set `APP_API_BASE_URL` to an empty/relative value and configure `BACKEND_API_HOST` on the frontend.

### Why is OBO required for Fabric Data Agent questions?

Fabric Data Agent queries execute with the signed-in user's identity. The API must use the Microsoft Entra On-Behalf-Of (OBO) flow; managed identity is not a substitute for this delegated Fabric access path. Follow [Set Up OBO Authentication](./SetupOBOAuthentication.md).

### What does the OBO setup script configure?

It creates or configures a shared app registration, a delegated `user_impersonation` scope, API permissions and consent, EasyAuth on both App Services, token storage, and OBO settings on the API. Authentication changes can take up to 10 minutes to propagate.

### A signed-in user sees “An error occurred. Please try again later.” How do I diagnose it?

Download the API App Service logs and correlate the request timestamp. Check, in order:

1. The frontend forwarded an access token.
2. The API selected an OBO credential rather than a managed identity fallback.
3. The user has Fabric workspace access.
4. The user has Azure AI Foundry data-plane permissions.

Avoid relying on the browser message alone; the API log identifies the downstream service and denied action.

### The API log says the user lacks `Microsoft.CognitiveServices/accounts/AIServices/agents/write`. How do I fix it?

Assign **Foundry User** to the user at the Foundry project scope. If project-scope assignment is unavailable in your environment, assign it at the Foundry account scope. This Foundry data-plane role grants the agent actions required to create conversations and runs.

`Owner`, `Azure AI Developer`, and `Cognitive Services OpenAI User` do not by themselves grant the required Foundry Agent data actions.

### Which Foundry role should an application user receive?

Use **Foundry User** for users who need to build or test against the project and run the agent workflow. Use **Foundry Agent Consumer** only when endpoint-consumption permissions are sufficient. Use **Foundry Project Manager** or **Foundry Owner** only for users who need project-management or broader administrative responsibilities.

### A user received a new role but still gets 403. What now?

Wait for Azure RBAC propagation, then refresh the browser session or sign out and in again. Recheck the role assignment scope and principal object ID before changing application code.

## Access management

### What access does a lab administrator usually need?

A lab administrator normally needs Azure resource management access, Fabric capacity and workspace administration, Foundry project access, and the relevant data-plane roles for Search, Storage, Cosmos DB, and ACR. Assign only roles needed for the person’s responsibilities outside a lab environment.

### Does subscription Owner grant access to all data planes?

No. Azure management-plane ownership does not automatically grant data-plane access to services such as Azure AI Search, Storage, Cosmos DB, Fabric, or Foundry agents. Assign their service-specific roles explicitly.

### What roles are commonly needed to initialize the sample data?

Depending on the operation, the initializing identity may need:

| Service | Typical role |
|---|---|
| Azure AI Search | Search Index Data Contributor |
| Storage | Storage Blob Data Contributor |
| Cosmos DB | Cosmos DB Built-in Data Contributor |
| Azure Container Registry | AcrPush |
| Fabric workspace | Admin or Contributor |
| Foundry project | Foundry User or a broader Foundry project role |

## Operations and recovery

### How do I confirm that the application is healthy?

Verify both App Service endpoints return successfully, then test:

1. A simple non-Fabric chat message.
2. A Fabric data question, such as revenue by year.
3. A search-grounded question, if a knowledge base is configured.

For authenticated requests, confirm API logs show user-token forwarding, OBO credential selection, and successful streaming completion.

### What should I collect before reporting a problem?

Collect the approximate UTC timestamp, the signed-in user, the exact prompt, the browser response, and the relevant API App Service log entries. Redact access tokens, client secrets, connection strings, and other credentials before sharing logs.

### Should generated scenario configuration be committed?

No. Generated scenario configuration can contain environment-specific resource IDs and deployment artifacts. Keep it out of source control and document only redacted, reusable deployment guidance.

### How do I delete the lab?

Delete the resource group only when you no longer need the deployment and have confirmed that no shared resources are being reused. See [Delete Resource Group](./DeleteResourceGroup.md).

## Related documentation

- [Deployment Guide](./DeploymentGuide.md)
- [Fabric Deployment](./Fabric_deployment.md)
- [Set Up OBO Authentication](./SetupOBOAuthentication.md)
- [Azure Account Set Up](./AzureAccountSetUp.md)
- [Prerequisites and Costs](./PrerequisitesCosts.md)
- [Security Guidelines](./SecurityGuidelines.md)
