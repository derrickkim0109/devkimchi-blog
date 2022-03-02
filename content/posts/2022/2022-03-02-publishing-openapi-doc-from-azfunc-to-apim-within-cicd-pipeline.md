---
title: "Publishing OpenAPI Document from Azure Functions to Azure API Management within CI/CD Pipeline"
slug: publishing-openapi-doc-from-azfunc-to-apim-within-cicd-pipeline
description: "In this post, I'm going to discuss how to publish the OpenAPI doc automatically generated from the Azure Function app, within the CI/CD pipeline."
date: "2022-03-02"
author: Justin-Yoo
tags:
- azure
- azure-functions
- openapi
- azure-api-management
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2022/03/publishing-openapi-doc-from-azfunc-to-apim-within-cicd-pipeline-00.png
fullscreen: true
---

In my [previous post][post 1], we've walked through how to run the [Azure Functions app][az fncapp] as a background process and locally generate the OpenAPI document from it within the [GitHub Actions workflow][gha]. The OpenAPI document can be stored as an artifact and integrated with [Power Platform][pp] or [Azure API Management (APIM)][az apim] later on.

* [Generating OpenAPI Document from Azure Functions within CI/CD Pipeline][post 1]
* ***Publishing OpenAPI Document from Azure Functions to Azure API Management within CI/CD Pipeline*** 👈

Throughout this post, I'm going to discuss how to publish the OpenAPI document to [APIM][az apim], after locally running the Azure Functions app as a background process and generating the OpenAPI document within the [GitHub Actions workflow][gha].

> **NOTE**: Let's use the sample app provided by the [Azure Functions OpenAPI Extension][az fncapp ext openapi] repository.


## Update GitHub Actions Workflow ##

The GitHub Actions workflow built in my [previous post][post 1] looks like the following (line #21-35). Let's assume that we use the Ubuntu runner and PowerShell.

```yaml
name: Build

on:
  push:

jobs:
  build_and_test:
    name: Build
    runs-on: 'ubuntu-latest'

    steps:
    - name: Build solution
      shell: pwsh
      run: |
        pushd MyFunctionApp

        dotnet build . -c Release -v minimal

        popd

    - name: Generate OpenAPI document
      shell: pwsh
      run: |
        cd MyFunctionApp

        Start-Process -NoNewWindow func @("start","--verbose","false")
        Start-Sleep -s 60

        Invoke-RestMethod -Method Get -Uri http://localhost:7071/api/swagger.json | ConvertTo-Json -Depth 100 | Out-File -FilePath outputs/swagger.json -Force

        Get-Content -Path outputs/swagger.json

        cd ..
```

The OpenAPI document generated from the pipeline above has `https://localhost:7071/api` as the default server URL. However, the actual server URL must be the format of `https://<azure-functions-app>.azurewebsites.net/api` after deployment. Therefore, although you haven't deployed the function app yet, the generated OpenAPI document must have the deployed app URL. According to the [doc][az fncapp ext openapi doc], the extension offers the feature to apply the server URL.

* Set the environment variable, `OpenApi__HostNames`, so the actual server URL is added on top of `localhost` (line #4).
* Set the environment variable, `AZURE_FUNCTIONS_ENVIRONMENT`, to `Production` so that `localhost` is omitted and only the server URL is rendered as if the function app is running on Azure (line #5).
* Generally speaking, `local.settings.json` is excluded from the repository because it's supposed to use for local development. But, it's required to run the function app locally as a background process. Therefore, it's always a good idea to create one. The revised action below copies the `local.settings.sample.json` to `local.settings.json` (line #12).

With these three points in mind, let's update the GitHub Action (line #4,5,12).

```yaml
    - name: Generate OpenAPI document
      shell: pwsh
      env:
        OpenApi__HostNames: 'https://<azure-functions-app>.azurewebsites.net/api'
        AZURE_FUNCTIONS_ENVIRONMENT: 'Production'
      run: |
        cd MyFunctionApp

        mkdir outputs

        # Create local.settings.json
        cp ./local.settings.sample.json ./local.settings.json

        Start-Process -NoNewWindow func @("start","--verbose","false")
        Start-Sleep -s 60

        Invoke-RestMethod -Method Get -Uri http://localhost:7071/api/swagger.json | ConvertTo-Json -Depth 100 | Out-File -FilePath outputs/swagger.json -Force

        Get-Content -Path outputs/swagger.json -Raw

        cd ..
```

As mentioned above, this action just copies the existing `local.settings.sample.json` file to `local.settings.json`, but it's not common. Therefore, in most cases, you should create the `local.settings.json` file by yourself, and it MUST include the environment variable, `FUNCTIONS_WORKER_RUNTIME` (line #3). In other words, the minimal structure of `local.settings.json` looks like below (assuming you use the in-proc worker):

```json
{
  "Values": {
    "FUNCTIONS_WORKER_RUNTIME": "dotnet"
  }
}
```

Finally, you will see the server URL applied from the OpenAPI document (line #4,16).

```json
// OpenAPI v2
{
  "swagger": "2.0",
  "host": "<azure-functions-app>.azurewebsites.net",
  "basePath": "/api",
  "schemes": [
    "https"
  ]
}

// OpenAPI v3
{
  "openapi": "3.0.1",
  "servers": [
    {
      "url": "https://<azure-functions-app>.azurewebsites.net/api"
    }
  ]
}
```

Now, we've got the OpenAPI document from the CI/CD pipeline, with the actual server URL, without deploying the function app.


## Publish to Azure API Management ##

This time, let's publish the OpenAPI document to [APIM][az apim]. Although there are many ways to do it, let's use [Azure Bicep][az bicep] and deploy it through [Azure CLI][az cli] in this post. Here's the first part of the bicep file. Use the [`existing` keyword][az bicep existing] to get the existing APIM resource details

```javascript
// azuredeploy.bicep
param servicename string

resource apim 'Microsoft.ApiManagement/service@2021-08-01' existing = {
    name: servicename
    scope: resourceGroup(apiManagement.groupName)
}
```

Then, with these APIM details, declare the API resource.

```javascript
// azuredeploy.bicep
param openapidoc string

resource apimapi 'Microsoft.ApiManagement/service/apis@2021-08-01' = {
    name: '${apim.name}/my-api'
    properties: {
        type: 'http'
        displayName: 'My API'
        description: 'This is my API.'
        path: 'myapi'
        subscriptionRequired: true
        format: 'openapi+json-link'
        value: openapidoc
    }
}
```

It looks simple, except those two attributes &ndash; `format` and `value` (line #12-13). Therefore, it's better to understand what those two attributes are. For more comprehensive details, you can visit the [Azure Bicep Template Reference][az apim bicep template].

1. TO use the JSON string parsed from the OpenAPI document:
   * **OpenAPI v2**: Set the `format` value to `swagger-json`.
   * **OpenAPI v3**: Set the `format` value to `openapi+json`.
   * Assign the JSON string to the `value` attribute.
2. TO use the publicly accessible URL for the OpenAPI document:
   * **OpenAPI v2**: Set the `format` value to `swagger-link-json`.
   * **OpenAPI v3**: Set the `format` value to `openapi+json-link`.
   * Assign the publicly accessible URL to the `value` attribute.

Even if the first approach is doable, I wouldn't recommend it because it's too painful to parse the OpenAPI document to the JSON string that the bicep file can understand. Therefore, I would suggest using the second approach by storing the OpenAPI document to [Azure Blob Storage][az st blob] right after it's generated from the pipeline.

Once you complete authoring the bicep file, run the Azure CLI command to deploy the API to APIM.

```powershell
$servicename = "<my_apim_service_name>"
$openapidoc = https://<my_blob_storage_name>.blob.core.windows.net/<container>/openapi.json

az deployment group create `
    -g <resource_group_name> `
    -n <deployment_name> `
    -f ./azuredeploy.bicep `
    -p servicename=$servicename `
    -p openapidoc=$openapidoc
```

You can confirm that the API is correctly deployed to APIM on Azure Portal.

---

So far, we've walked through how:

1. To run [Azure Functions app][az fncapp] as a background process,
1. To generate the OpenAPI document from the app by applying the deployed server URL without actually deploying it, and
1. To publish the OpenAPI document to [APIM][az apim] within [GitHub Actions workflow][gha].

By doing so, you will be able to reduce some maintenance overheads by automating the integration part.


[post 1]: /2022/02/23/generating-openapi-document-from-azure-functions-within-cicd-pipeline/
[post 2]: /2022/03/02/publishing-openapi-doc-from-azfunc-to-apim-within-cicd-pipeline/

[pp]: https://powerplatform.microsoft.com/?WT.mc_id=dotnet-59289-juyoo

[az fncapp]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-59289-juyoo
[az fncapp ext openapi]: https://aka.ms/azfunc-openapi
[az fncapp ext openapi doc]: https://github.com/Azure/azure-functions-openapi-extension/blob/main/docs/openapi.md#configure-custom-base-urls

[az apim]: https://docs.microsoft.com/azure/api-management/api-management-key-concepts?WT.mc_id=dotnet-59289-juyoo
[az apim bicep template]: https://docs.microsoft.com/azure/templates/microsoft.apimanagement/service/apis?WT.mc_id=dotnet-59289-juyoo&tabs=bicep#apicreateorupdateproperties

[az bicep]: https://docs.microsoft.com/azure/azure-resource-manager/bicep/overview?tabs=bicep&WT.mc_id=dotnet-59289-juyoo
[az bicep existing]: https://docs.microsoft.com/azure/azure-resource-manager/bicep/existing-resource?WT.mc_id=dotnet-59289-juyoo

[az st blob]: https://docs.microsoft.com/azure/storage/blobs/storage-blobs-introduction?WT.mc_id=dotnet-59289-juyoo

[az cli]: https://docs.microsoft.com/cli/azure/what-is-azure-cli?WT.mc_id=dotnet-59289-juyoo

[gha]: https://github.com/features/actions

[powershell]: https://docs.microsoft.com/powershell/scripting/overview?WT.mc_id=dotnet-59289-juyoo
