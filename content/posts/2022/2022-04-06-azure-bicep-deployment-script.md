---
title: "Azure Apps Autopilot #2 - Deployment Script"
slug: azure-bicep-deployment-script
description: "In this post, I'm going to introduce Azure Bicep DeploymentScripts so that the autopilot feature is completely available through ARM."
date: "2022-04-06"
author: Justin-Yoo
tags:
- azure
- autopilot
- bicep
- developer-experience
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2022/04/azure-bicep-deployment-script-00.png
fullscreen: true
---

In my [previous post][post 1], we built the autopilot feature, using various event triggers from [GitHub Actions][gh actions] and [Azure Bicep][az bicep]. Throughout this post, I'm going to build the revised autopilot feature using [Azure Bicep Deployment Scripts resource][az bicep ds] without GitHub Actions.

> You can download the sample code from [this GitHub repository][gh sample].

## Solution Architecture ##

Let's say you're building a microservices architecture. It typically consists of an API gateway and many API apps, which are [Azure API Management (APIM)][az apim] and [Azure Functions][az fncapp] API in this example. The microservices architecture might be more complex depending on the requirements, but it's way too far from our topic. Therefore, let's build a minimal structure that is working. Here's the diagram.

![Microservices Architecture on Azure][image-01]

## Azure Resources Provisioning ##

There are five resources to provision:

* [Azure Storage][az st] 👉 [View Bicep code][gh sample st]
* [Application Insights][az appins] 👉 [View Bicep code][gh sample appins]
* [App Service Plan][az csplan] 👉 [View Bicep code][gh sample csplan]
* [Function App][az fncapp] 👉 [View Bicep code][gh sample fncapp]
* [Azure API Management][az apim] 👉 [View Bicep code][gh sample apim]

As you can see, each resource has its corresponding Bicep module. For example, it's required to provision Azure Storage, Application Insights and App Service Plan (Consumption) before creating an Azure Functions app. Therefore, the [`provision_functionapp.bicep`][gh sample fncapp provision] file takes care of this orchestration. In addition to that, Azure API Management also needs Application Insights as a dependency, so the [`provision_apimamagement.bicep`][gh sample apim provision] file looks after this orchestration.

After deploying the function app, the [`provision_apimanagementapi.bicep`][gh sample apim api provision] registers the function app to APIM.

> **NOTE**: I used the modularisation approach while writing the Bicep files. But it may differ from your situation, and writing one big Bicep file could be more efficient in some circumstances.

Once you complete writing the Bicep file, you might expect the following processes in that order.

1. Provision the function related resources 👉 [`provision_functionapp.bicep`][gh sample fncapp provision]
2. Provision the APIM related resources 👉 [`provision_apimamagement.bicep`][gh sample apim provision]
3. ➡️ Deploy the function app ⬅️
4. Integrate the function app with APIM 👉 [`provision_apimanagementapi.bicep`][gh sample apim api provision]

When #1 and #2 are over, you can see the following resources provisioned.

![Azure Resource Provisioning][image-02]

By the way, #4 can't be provisioned until the function app is deployed at #3. Moreover, #3 is not related to resource provisioning but app deployment. What if we can convert this app deployment experience into resource provisioning? Then, all processes from #1 to #4 can be done within the resource provisioning pipeline, which means the whole "autopilot" feature is set.

## Azure Bicep &ndash; Deployment Scripts ##

[Azure Resource Manager (ARM)][az rm] has introduced the concept of [deployment script][az bicep ds]. Through this deployment script, ARM can include PowerShell scripts or bash scripts as a part of the resource provisioning pipeline. In other words, the deployment script resource can run [Azure PowerShell][az pwsh] or [Azure CLI][az cli]. How is that possible? Here's the Bicep file declaring the deployment script.

```javascript
resource ds 'Microsoft.Resources/deploymentScripts@2020-10-01' = {
    name: 'my-deployment-script'
    location: resourceGroup().location
    kind: 'AzureCLI'
    identity: {
        type: 'UserAssigned'
        userAssignedIdentities: {
            '<user-assigned-identity-id>': {}
        }
    }
    properties: {
        azCliVersion: '2.33.1'
        containerSettings: {
            containerGroupName: 'my-container-group'
        }
        environmentVariables: [
            {
                name: 'RESOURCE_NAME'
                value: '<resource-name>'
            }
        ]
        primaryScriptUri: '<bash-script-url>'
        retentionInterval: 'P1D'
    }
}
```

* The `kind` attribute explicitly declares that it uses Azure CLI. You can declare "AzurePowerShell" if you like.
* The bash script needs the login credentials to run Azure CLI commands. Therefore, it uses the user-assigned identity.
* The bash script uses the Azure CLI of `v2.33.1` in this example. It's recommended to use a specific version instead of the latest version, discussed later.
* The bash script uses the environment variable of `RESOURCE_NAME`, which is declared and has the value assigned in the Bicep file.
* The `primaryScriptUri` attribute declares the publicly accessible bash script URL.

> **NOTE**: The Bicep file only shows the necessary bits and pieces. If you want to know more details, please take a look at [this Bicep file][gh sample ds].

So, what does this deployment script resource do? It temporarily provisions both [Azure Container Instance][az aci] and [Storage Account][az st] to handle the script. According to [this document][az bicep ds], using the Azure CLI version older than 30 days from the day running the script is recommended. Therefore, at the time of this writing, [`v2.33.1`][az cli release notes] is the closest one. Of course, you can use the `az upgrade` command within the script to get the latest version, but it's totally up to you.

## Deployment Script &ndash; Bash Script ##

Take a look at the bash script that runs the series of [Azure CLI][az cli] commands. Let's say the artifact name is `api.zip`, and the following script gets the artifact URL stored in the GitHub repository at the beginning.

```powershell
# Get artifacts from GitHub
urls=$(curl -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/devkimchi/APIM-OpenAPI-Sample/releases/latest | \
  jq '.assets[] | { name: .name, url: .browser_download_url }')

apizip=$(echo $urls | jq 'select(.name == "api.zip") | .url' -r)
```

As the artifact URL is set as the environment variable, `$apizip`, Azure CLI uses this artifact URL to deploy the function app. The environment variable, `$RESOURCE_NAME`, comes from the deployment script Bicep file.

```powershell
# Deploy function apps
ipapp=$(az functionapp deploy \
  -g rg-$RESOURCE_NAME \
  -n fncapp-$RESOURCE_NAME-api \
  --src-url $apizip \
  --type zip)
```

Once we complete deployment, it should be registered to APIM through the OpenAPI document. How can we do that? Another Bicep file, `provision-apimanagementapi.bicep`, can also be run within the bash script through Azure CLI. But make sure that, if you want to provision resources through URL, you MUST convert the Bicep file to an ARM template of the JSON type.

```powershell
# Provision APIs to APIM
az deployment group create \
  -n ApiManagement_Api \
  -g rg-$RESOURCE_NAME \
  # MUST be ARM template, not Bicep
  -u https://raw.githubusercontent.com/devkimchi/APIM-OpenAPI-Sample/main/Resources/provision-apimanagementapi.json \
  -p name=$RESOURCE_NAME
```

Now, we've got the bash script to run within the deployment script resource.

## Overall Resource Orchestration ##

As the final step, everything written above should be composed into one. Here's the [`main.bicep`][gh sample main] file that orchestrates resources and deployment scripts. The deployment script resource should always come last after all other resources are provisioned by using the `dependsOn` attribute.

```javascript
// Provision API Management
module apim './provision-apimanagement.bicep' = {
    name: 'ApiManagement'
    params: {
        ...
    }
}

// Provision function app
module fncapp './provision-functionapp.bicep' = {
    name: 'FunctionApp'
    dependsOn: [
        apim
    ]
    params: {
        ...
    }
}]

// Provision deployment script
module uai './deploymentScript.bicep' = {
    name: 'UserAssignedIdentity'
    dependsOn: [
        apim
        fncapp
    ]
    params: {
        ...
    }
}
```

Once everything is done, convert the last Bicep file into the ARM template and link it to the image button below. Then, just click the button and put in the necessary information. It will automatically provision resources, deploy apps and do the rest of the provisioning process within one single pipeline, and the app is ready to use.

[![Deploy To Azure](https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/deploytoazure.svg?sanitize=true)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fdevkimchi%2FAPIM-OpenAPI-Sample%2Fmain%2FResources%2Fazuredeploy.json)

If you actually click the button above, you will be able to see the Azure Portal screen like below:

![Azure Portal Custom Template Provisioning][image-03]

Did you complete all the steps? Then go to [APIM][az apim] and visit one of the function app's Swagger UI endpoint.

![Azure Functions Swagger UI through APIM][image-04]

---

So far, we've walked through the [Azure Bicep][az bicep]'s [deployment script][az bicep ds] resources and revised the autopilot feature without needing to rely on [GitHub Actions][gh actions], as mentioned in the [previous post][post 1]. Now, you can hand over your repository to your sales representatives or other dev units to take a look with no knowledge of your application set-up. How easy is that?


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/04/azure-bicep-deployment-script-01-en.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/04/azure-bicep-deployment-script-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/04/azure-bicep-deployment-script-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/04/azure-bicep-deployment-script-04.png


[post 1]: /2022/03/11/azure-apps-autopilot/
[post 2]: /2022/04/06/azure-bicep-deployment-script/

[az aci]: https://docs.microsoft.com/azure/container-instances/container-instances-overview?WT.mc_id=62548-dotnet-juyoo
[az apim]: https://docs.microsoft.com/azure/api-management/api-management-key-concepts?WT.mc_id=62548-dotnet-juyoo
[az appins]: https://docs.microsoft.com/azure/azure-monitor/app/app-insights-overview?WT.mc_id=62548-dotnet-juyoo

[az bicep]: https://docs.microsoft.com/azure/azure-resource-manager/bicep/overview?tabs=bicep&WT.mc_id=62548-dotnet-juyoo
[az bicep ds]: https://docs.microsoft.com/azure/azure-resource-manager/bicep/deployment-script-bicep?WT.mc_id=62548-dotnet-juyoo

[az cli]: https://docs.microsoft.com/cli/azure/what-is-azure-cli?WT.mc_id=62548-dotnet-juyoo
[az cli release notes]: https://docs.microsoft.com/cli/azure/release-notes-azure-cli?WT.mc_id=62548-dotnet-juyoo

[az csplan]: https://docs.microsoft.com/azure/app-service/overview-hosting-plans?WT.mc_id=62548-dotnet-juyoo
[az fncapp]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=62548-dotnet-juyoo
[az rm]: https://docs.microsoft.com/azure/azure-resource-manager/management/overview?WT.mc_id=62548-dotnet-juyoo
[az st]: https://docs.microsoft.com/azure/storage/common/storage-account-overview?WT.mc_id=62548-dotnet-juyoo
[az pwsh]: https://docs.microsoft.com/powershell/azure/what-is-azure-powershell?WT.mc_id=62548-dotnet-juyoo

[gh sample]: https://github.com/devkimchi/APIM-OpenAPI-Sample
[gh sample st]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/storageAccount.bicep
[gh sample appins]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/appInsights.bicep
[gh sample csplan]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/consumptionPlan.bicep
[gh sample fncapp]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/functionApp.bicep
[gh sample apim]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/apiManagement.bicep
[gh sample fncapp provision]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/provision-functionapp.bicep
[gh sample apim provision]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/provision-apimanagement.bicep
[gh sample apim api provision]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/provision-apimanagementapi.bicep
[gh sample ds]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/deploymentScript.bicep
[gh sample main]: https://github.com/devkimchi/APIM-OpenAPI-Sample/blob/main/Resources/main.bicep

[gh actions]: https://github.com/features/actions
