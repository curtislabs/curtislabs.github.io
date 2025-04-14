---
layout: post
title: "Remove Azure API Version Constraint"
date: 2024-06-22T05:00:00Z
authors: ["Adam Curtis"]
categories: ["Azure", "API Management"]
description: "How to remove an Azure API Gateway version constraint using the Azure CLI."
thumbnail: "/assets/images/gen/blog/azure-api-gateway-logo-thumbnail.png"
# image: "/assets/images/gen/blog/azure-api-gateway-logo.png"
---
Microsoft [recently shared](https://learn.microsoft.com/en-us/azure/api-management/breaking-changes/api-version-retirement-sep-2023) they were retiring Azure Resource Manager API versions, and as part of that, users would need to set a minimum API version in Azure API Management. I [followed their directions](https://learn.microsoft.com/en-us/azure/api-management/breaking-changes/api-version-retirement-sep-2023#update-minimum-api-version-setting-on-your-api-management-instance) to set our API version to **2021-08-01**, but then noticed all of our connections to Azure API Management from Logic Apps were broken with the error *"API version query parameter is not specified or was specified incorrectly"*. 

The only way to fix this error is to undo the API versioning using the Azure CLI. To fix this, you can run the following commands in PowerShell and substitute the ID placeholders with your own.

### Powershell

```
winget install -e --id Microsoft.AzureCLI --accept-source-agreements
az login
az account list
az account set --subscription <subscription_id>
az apim list
az apim update -n <api_management_name> -g <resource_group_name> --remove apiVersionConstraint
```
