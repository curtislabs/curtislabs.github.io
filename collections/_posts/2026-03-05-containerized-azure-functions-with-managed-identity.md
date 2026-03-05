---
layout: post
title: "Containerized Azure Functions + Microsoft Graph with Managed Identity (Azure Container Apps Guide)"
date: 2026-03-05T05:00:00Z
authors: ["Adam Curtis"]
categories: ["Azure", "Azure Functions", "Containers", "Microsoft Graph", "Identity"]
description: "Learn how to run a containerized C# Azure Function in Azure Container Apps that securely calls Microsoft Graph using a user-assigned managed identity. This guide shows how to eliminate secrets using Azure Storage identity access, Key Vault for function keys, and managed identity authentication."
thumbnail: "/assets/images/gen/blog/azure-functions-container-logo-thumbnail.png"
# image: "/assets/images/gen/blog/azure-functions-container-logo.png"
---

# Running a Containerized C# Azure Function in Azure Container Apps with Microsoft Graph and Managed Identity

I recently needed to build a small API that could query Microsoft Graph without user interaction, no login prompts, no OAuth flows, and definitely no secrets in configuration files. The goal was simple: a containerized Azure Function running in Container Apps that could authenticate to Microsoft Graph using a managed identity.

If you've ever tried to wire up all the pieces, storage authentication, Key Vault access, Graph permissions, and container configuration, you know there are about a dozen ways it can go wrong. This guide walks through the entire setup, from creating the infrastructure to deploying a working containerized function.

------------------------------------------------------------------------

## Setting Up Your Environment

Before we dive into creating resources, you'll need a couple of PowerShell modules installed. We'll be using both the Az module for Azure resource management and the Microsoft.Graph module for configuring Graph permissions.

``` powershell
Install-Module Az -Scope CurrentUser
Install-Module Microsoft.Graph -Scope CurrentUser
```

Next, let's define all the variables we'll need throughout this process. Having these upfront makes the rest of the commands cleaner and easier to modify for your environment.

``` powershell
$tenant = '<tenant_id>'
$subscription = '<subscription_id>'

$rgName = '<resource_group_name>'
$location = '<azure_datacenter_location>'

$storageName = '<storage_accountname>'
$storageSku = 'Standard_LRS'
$storageKind = 'StorageV2'

$kvName = '<globally_unique_key_vault_name>'
$kvSku = 'Standard'

$uamiName = '<user_assigned_managed_identity_name>'

$acr='<azure_container_registry_name>'
$image='<image_name>'
$tag='<image_tag_name>'

$functionAppEnvironmentName = '<function_app_environment_name>'
$functionAppName = '<function_container_app_name>'
```

Now connect to all the services we'll be working with, Azure Resource Manager, Azure CLI, Microsoft Graph, and your container registry:

``` powershell
Connect-AzAccount -Tenant $tenant -Subscription $subscription
az login --tenant $tenant
Connect-MgGraph -Scopes "Application.Read.All","AppRoleAssignment.ReadWrite.All","Directory.Read.All" -NoWelcome
az acr login -n $acr
```

## Building the Azure Infrastructure

With our environment configured, let's create the Azure resources that will support our function. We'll start with the basics: a resource group, storage account, and Key Vault.

``` powershell
# Create the resource group
$rg = New-AzResourceGroup -Name $rgName -Location $location

# Create storage account for the Functions runtime
$storage = New-AzStorageAccount -ResourceGroupName $rgName -Name $storageName -Location $location -SkuName $storageSku -Kind $storageKind

# Create Key Vault for storing function keys securely
$kv = New-AzKeyVault -Name $kvName -ResourceGroupName $rgName -Location $location -Sku $kvSku 
```

The storage account is critical, the Azure Functions runtime uses it for internal state management, triggers, and bindings. The Key Vault will store our function keys so we don't have to rely on the default file-based storage (which doesn't work with ephemeral containers).

## Configuring the Managed Identity

We'll create a user-assigned managed identity and grant it all the permissions it needs. This single identity will be used by three different components: your function code (to call Graph), the Functions runtime (to access storage), and the runtime again (to read secrets from Key Vault).

``` powershell
# Create the managed identity
$uami = az identity create --resource-group $rgName --name $uamiName | ConvertFrom-Json
```

Now we need to grant this identity permission to call Microsoft Graph. We're setting up application permissions here (not delegated permissions), which is important because there's no user context, the function runs as itself.

``` powershell
# Get the Microsoft Graph service principal
$graphSp = Get-MgServicePrincipal -Filter "appId eq '00000003-0000-0000-c000-000000000000'"

# Find the User.Read.All application role
$role = $graphSp.AppRoles | Where-Object {
  $_.Value -eq "User.Read.All" -and $_.AllowedMemberTypes -contains "Application"
}

# Grant the permission to our managed identity
New-MgServicePrincipalAppRoleAssignment -ServicePrincipalId $uami.principalId -PrincipalId $uami.principalId -ResourceId $graphSp.Id -AppRoleId $role.Id
```

The managed identity also needs Azure RBAC roles to access storage and Key Vault. These permissions allow the Functions runtime to authenticate without connection strings:

``` powershell
New-AzRoleAssignment -ObjectId $uami.PrincipalId -RoleDefinitionName "Storage Blob Data Contributor" -Scope $storage.Id
New-AzRoleAssignment -ObjectId $uami.PrincipalId -RoleDefinitionName "Storage Queue Data Contributor" -Scope $storage.Id
New-AzRoleAssignment -ObjectId $uami.PrincipalId -RoleDefinitionName "Key Vault Secrets Officer" -Scope $kv.ResourceId
```

## Creating the Azure Function

Let's build the actual function. We'll use the .NET isolated worker model, which gives us better dependency injection and the ability to use the latest .NET features.

``` powershell
func init GraphFunctionApp --worker-runtime dotnet-isolated
cd GraphFunctionApp

dotnet add package Azure.Identity
dotnet add package Microsoft.Graph
```

The application settings are crucial. Notice we're using `AzureWebJobsStorage__accountName` instead of a connection string, this is how you tell the Functions runtime to use identity-based authentication for storage.

Create or update your `local.settings.json` with your specific values:

``` json
{
  "Values": {
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "ManagedIdentityClientId": "<client-id>",
    "AzureWebJobsSecretStorageType": "keyvault",
    "AzureWebJobsSecretStorageKeyVaultUri": "https://<vault>.vault.azure.net/",
    "AzureWebJobsStorage__accountName": "<storage-account>"
  }
}
```

Here's the most common mistake I see: people try to use `AzureWebJobsStorage` with a connection string when using managed identity. That won't work. You need the double-underscore syntax (`__accountName`) to signal identity-based auth.

## Wiring Up Microsoft Graph in Your Code

Now for the dependency injection setup in `Program.cs`. The code checks if we're running in development mode or if the managed identity client ID is missing. If either is true, it uses `DefaultAzureCredential`, which falls back to your local Azure CLI or Visual Studio credentials. In production, it uses the managed identity.

``` csharp
builder.Services.AddSingleton(sp =>
{
    var miClientId = Environment.GetEnvironmentVariable("ManagedIdentityClientId");

    TokenCredential cred =
        string.IsNullOrWhiteSpace(miClientId) ||
        Environment.GetEnvironmentVariable("AZURE_FUNCTIONS_ENVIRONMENT") == "Development"
            ? new DefaultAzureCredential()
            : new ManagedIdentityCredential(miClientId);

    var authProvider = new AzureIdentityAuthenticationProvider(
        cred,
        isCaeEnabled: true,
        scopes: ["https://graph.microsoft.com/.default"]);

    return new GraphServiceClient(GraphClientFactory.Create(), authProvider);
});
```

The `isCaeEnabled: true` setting enables Continuous Access Evaluation, which is an Azure AD feature that lets tokens be revoked in near-real-time. The scope `https://graph.microsoft.com/.default` tells Azure AD to grant all the application permissions we configured earlier.

## Writing Your Function

Here's an example function that queries Microsoft Graph for a user by their employee ID.

``` csharp
[Function("GetUserByEmployeeNumber")]
public async Task<HttpResponseData> Run(
    [HttpTrigger(AuthorizationLevel.Function, "get", Route = "employees/{employeeNumber}")]
    HttpRequestData req,
    string employeeNumber)
{
    var result = await _graph.Users.GetAsync(q =>
    {
        q.QueryParameters.Filter = $"employeeId eq '{employeeNumber.Replace("'", "''")}'";
        q.QueryParameters.Select = ["id","displayName","employeeId","mail"];
        q.QueryParameters.Top = 1;
    });

    var user = result?.Value?.FirstOrDefault();

    var res = req.CreateResponse(user is null ? HttpStatusCode.NotFound : HttpStatusCode.OK);

    if (user is null)
        await res.WriteAsJsonAsync(new { message = "User not found" });
    else
        await res.WriteAsJsonAsync(user);

    return res;
}
```

## Containerizing the Function

Azure Functions can run in containers, which gives you more control over the runtime environment and makes it easier to deploy to services like Container Apps. The Dockerfile is straightforward, a multi-stage build that compiles your function and then packages it into the Azure Functions runtime container.

``` dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /app/publish /p:UseAppHost=false

FROM mcr.microsoft.com/azure-functions/dotnet-isolated:4-dotnet-isolated10.0
WORKDIR /home/site/wwwroot
COPY --from=build /app/publish .

ENV FUNCTIONS_WORKER_RUNTIME=dotnet-isolated AzureWebJobsScriptRoot=/home/site/wwwroot AzureFunctionsJobHost__Logging__Console__IsEnabled=true ASPNETCORE_URLS=http://0.0.0.0:8080

EXPOSE 8080
```

The environment variables here are important. `ASPNETCORE_URLS` tells the function to listen on port 8080, which is what Container Apps expects by default. The logging settings ensure you can see what's happening in the container logs.

Build and push the image to your Azure Container Registry:

``` powershell
az acr build -r $acr -t "${image}:${tag}" .
```

The `az acr build` command is convenient, it builds the container in Azure rather than locally, which is faster if you have a slow upload speed or a large codebase.

## Deploying to Azure Container Apps

This command creates the container app, configures it to use your managed identity, and sets up the container registry authentication.

``` powershell
az containerapp create --name $functionAppName --resource-group $rgName --environment $functionAppEnvironmentName --image "$acr.azurecr.io/$image:$tag" --target-port 8080 --ingress external --user-assigned $uami.Id --registry-server "$acr.azurecr.io" --registry-identity $uami.Id
```


- `--user-assigned $uami.Id`: Assigns the managed identity to the container app
- `--registry-identity $uami.Id`: Tells Container Apps to use the same managed identity to authenticate to ACR
- `--ingress external`: Makes the function accessible from the internet
- `--target-port 8080`: Matches the port we exposed in the Dockerfile

## What Can Go Wrong (And How to Fix It)

The tricky part about containerized Azure Functions with managed identity is that you have three separate components all authenticating independently:

| Component         | Uses identity for     |
|-------------------|-----------------------|
| Function code     | Microsoft Graph       |
| Functions runtime | Azure Storage         |
| Functions runtime | Key Vault             |

If any one of these three isn't configured correctly, the container might start but the function won't work. The container health checks pass, but when you try to invoke the function, it fails.

The most common issue I've encountered is storage authentication. When using managed identity, you **must** configure the storage account name using the double-underscore syntax instead of a connection string:

```
AzureWebJobsStorage__accountName=<storage-account>
AzureWebJobsStorage__clientId=<identity-client-id>
```

If you use `AzureWebJobsStorage` with a connection string, the Functions runtime will try to use that instead of the managed identity, and you'll get authentication failures.

Also, the Microsoft Graph permissions must be **application permissions**, not delegated permissions. Delegated permissions require a user context, which doesn't exist when a function runs on its own. If you accidentally grant delegated permissions, your Graph calls will fail with cryptic authentication errors.

Finally, the function keys won't appear in Key Vault until you invoke the function for the first time. The Functions runtime generates these keys on-demand during the initial invocation and then stores them in Key Vault. After that first call, you'll find them in your vault as secrets.

## Wrapping Up

Once everything is wired up correctly, the container starts, the Functions runtime authenticates to storage and Key Vault using the managed identity, and your function code can call Microsoft Graph—all without storing a single secret or connection string anywhere.

The initial setup takes some effort, but the payoff is worth it. No more rotating secrets, no more worrying about connection strings leaking into logs, and no more credential management headaches. The managed identity handles everything, and Azure takes care of token refresh automatically.

If you're building Azure Functions that need to call Azure services or Microsoft Graph, this pattern is the way to go. It's more secure, easier to maintain, and aligns with Azure's zero-trust security model.
