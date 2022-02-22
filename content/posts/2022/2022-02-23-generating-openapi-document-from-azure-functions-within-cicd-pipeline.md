---
title: "Generating OpenAPI Document from Azure Functions within CI/CD Pipeline"
slug: generating-openapi-document-from-azure-functions-within-cicd-pipeline
description: "In this post, I'm going to discuss how to generate the OpenAPI document from an Azure Functions app within GitHub Actions workflow."
date: "2022-02-23"
author: Justin-Yoo
tags:
- azure
- azure-functions
- openapi
- github-actions
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2022/02/generating-openapi-document-from-azure-functions-within-cicd-pipeline-00.png
fullscreen: true
---

From time to time, you might be facing some situations while using the [Azure Functions OpenAPI extension][az fncapp ext openapi] for your [Azure Functions][az fncapp] development. One of them is that you are asked to generate the OpenAPI document from the function app right after the build and integrate it with [Power Platform][pp] or [Azure API Management][az apim]. What if you want to do this process within your CI/CD pipeline?

Throughout this post, I'm going to discuss how to generate the OpenAPI document within the GitHub Actions workflow right after the function app build.

> **NOTE**: Let's use the sample app provided by the [Azure Functions OpenAPI Extension][az fncapp ext openapi] repository.

First of all, you need to install the [Azure Functions Core Tools][az fncapp core tools] within your GitHub Actions workflow. Installing the tool varies depending on the runner OS. The command you're going to run the function app is like this:

https://gist.github.com/justinyoo/890cb4f3e409e237cf8405a7a343a04a?file=01-func-start.sh

If it's the terminal environment on your local machine, just open one terminal session and run the function app. After that, you can open another terminal session for additional command-line jobs. That's not a problem at all. However, it's a totally different story when it's about the CI/CD pipeline because you can't open multiple terminal sessions there. Therefore, you have to run the function app as a background process.


## Bash Shell Background Process ##

Bash shell offers `&` to run your app as the background process, like `func start &` (line #2). Therefore, you can run the shell commands to get the OpenAPI document like:

https://gist.github.com/justinyoo/890cb4f3e409e237cf8405a7a343a04a?file=02-run-function-app-in-bg.sh&highlights=2

How simple is that?


## PowerShell Background Process ##

If you prefer PowerShell, use the `Start-Process` cmdlet with the `-NoNewWindow` switch to run the function app as the background process (line #2). For example, here are the commands on **non-Windows** runner:

https://gist.github.com/justinyoo/890cb4f3e409e237cf8405a7a343a04a?file=03-run-function-app-in-bg-on-non-windows.ps1&highlights=2

On the other hand, the `Start-Process` cmdlet doesn't understand the command, `func` on the Windows runner. Therefore, you need a pre-process the `func` command (line #2, 5).

https://gist.github.com/justinyoo/890cb4f3e409e237cf8405a7a343a04a?file=04-run-function-app-in-bg-on-windows.ps1&highlights=2,5

Now, you can get the OpenAPI document in PowerShell.


## GitHub Actions Workflow ##

Now, we can get the OpenAPI document from the function app running as a background process. It means that you can get the document within the CI/CD pipeline. Here's the example in the GitHub Actions workflow. It only shows the function app build and OpenAPI document generation (line #26-33) for brevity.

https://gist.github.com/justinyoo/890cb4f3e409e237cf8405a7a343a04a?file=05-github-actions-workflow.yaml&highlights=26-33

---

So far, we've walked through how to generate the OpenAPI document from the function app running as a background process within the CI/CD pipeline. The generated document can be stored somewhere as an artifact and integrated with other services like [Power Platform][pp] or [Azure API Management][az apim].


[pp]: https://powerplatform.microsoft.com/?WT.mc_id=dotnet-58527-juyoo

[az fncapp]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-58527-juyoo
[az fncapp core tools]: https://docs.microsoft.com/azure/azure-functions/functions-run-local?WT.mc_id=dotnet-58527-juyoo
[az fncapp ext openapi]: https://aka.ms/azfunc-openapi

[az apim]: https://docs.microsoft.com/azure/api-management/api-management-key-concepts?WT.mc_id=dotnet-58527-juyoo
