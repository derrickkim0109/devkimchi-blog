---
title: "Azure Functions Integration Testing"
slug: azure-functions-integration-testing
description: "In this post, I'm going to discuss how to run a function app locally as a background process and implement it within GitHub Actions workflow."
date: "2021-08-26"
author: Justin-Yoo
tags:
- azure
- azure-functions
- integration-testing
- github-actions
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-integration-testing-00.png
fullscreen: true
---

A while ago, I wrote a blog post about [Azure Functions integration testing with Mountebank][post 1] and another blog post about [end-to-end (E2E) testing for Azure Functions][post 2]. In the post, I suggested deploying the [Azure Functions][az fncapp] app first before running the E2E testing. What if you can run the Azure Function app locally within the build pipeline? Then you can get the test result even before the app deployment, which may result in the fail-fast concept.

Throughout this post, I'm going to discuss how to run a function app locally within the build pipeline then run the integration testing scenarios instead of running the E2E testing after the app deployment.

> You can find the sample code used in this post at [this GitHub repository][gh sample].


## Simple Azure Functions App ##

Here's the straightforward Azure Function app code. I'm using the [Azure Functions OpenAPI extension][az fncapp openapi] in this app. It has only one endpoint like below:

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=01-default-http-trigger.cs

Run this function app and go to the URL, `http://localhost:7071/api/openapi/v3.json`, and you will see the following OpenAPI document.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=02-openapi-v3.json

At this stage, my focus is to make sure whether the OpenAPI document is correct or not. As you can see the OpenAPI document above, the document has a very simple data structure under the `components.schemas.greeting` node. What if the data type is complex? We need confirmation. In this case, should we deploy the app to Azure and run the endpoint over there? Maybe or maybe not.

But this time, let's run the app in the build pipeline and test it there, rather than deploying it to Azure.


## Running Azure Functions App as a Background Process ##

In order to test the Azure Functions app locally, it should be running on the local machine, using the [Azure Functions CLI][az fncapp cli].

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=03-func-start.sh

But the issue of this CLI doesn't offer a way to run the app as a background process, something like `func start --background`. Therefore, instead of relying on the CLI, we should use the shell command. So, for example, if you use the bash shell, run this `func start &` command first, then run `bg`.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=04-func-start-bg.sh

If you use PowerShell, use the `Start-Process` cmdlet with the `-NoNewWindow` switch so that the function app runs as a background process.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=05-func-start.ps1

Once the function app is running, execute this bash command on the same console session, `curl http://localhost:7071/api/openapi/v3.json`. Alternatively, use the PowerShell cmdlet `Invoke-RestMethod -Method Get -Uri http://localhost:7071/api/openapi/v3.json` to get the OpenAPI document.


## Writing Integration Test Codes ##

As we've got the function app running in the background, we can now write the test codes on top of that. So let's have a look at the code below. First, it sends a GET request to the endpoint, `http://localhost:7071/api/openapi/v3.json`, gets the response as a string and deserialises it to `OpenApiDocument`, and finally asserts the results whether it's expected or not.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=06-default-http-trigger-tests.cs

As you can see from the test codes above, there's no mocking. Instead, we just use the actual endpoint running on the local machine.

Once the test codes are ready, run the following command:

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=07-dotnet-test.sh

You will get the test results.


## Putting Altogether to GitHub Actions ##

We knew how to run the function app as a background process and got the test codes. Our CI/CD pipeline should be able to execute this. Let's have a look at the [GitHub Actions][gha] workflow. Some actions are omitted for brevity.


### GitHub-hosted Runners ###

It really depends on the situation, but I'm assuming we should test the code on all the operating systems &ndash; Windows, Mac and Linux. In this case, use the `matrix` attribute.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=08-build-1.yaml&highlights=5-6

GitHub-hosted runners don't have the [Azure Functions CLI][az fncapp cli] installed by default. Therefore, you should install it by yourself.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=08-build-2.yaml&highlights=8

Install the .NET Core 3.1 SDK as well.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=08-build-3.yaml

After all the tools are installed, testing should be followed.

The following action is for Mac and Linux runners. The first step is to run the function app as a background process. Although you can simply use the command `func start`, this action declares `func @("start","--verbose","false")` to minimise the noise from the local debugging log messages.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=08-build-4.yaml&highlights=2,10

On the other hand, the following action is for Windows runner. The main difference from the other action is that this time doesn't use the `func` command but the `$func` variable. Using the `func` command will get the error like `This command cannot be run due to the error: %1 is not a valid Win32 application.`. It's because the `func` command points to the `func.ps1` file, which is the PowerShell script. Instead of using this PowerShell script, you need to call `func.cmd` to run the function app as a background process.

https://gist.github.com/justinyoo/d00aefe95ff7a5292121fc06b604cfad?file=08-build-5.yaml&highlights=2,8,11

Once all the GitHub Actions workflow is set, push your codes to GitHub. Then you'll see all the build pipeline works as expected.

![Build Pipeline on Windows Runner][image-01]

![Build Pipeline on Non-Windows Runner][image-02]

---

So far, we've walked through how to run the integration testing codes for [Azure Functions app][az fncapp] within the [GitHub Actions][gha] workflow, using [Azure Functions CLI][az fncapp cli] as a background process. As a result, you can now avoid extra steps for the app deployment to Azure for testing.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-integration-testing-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-integration-testing-02.png


[post 1]: /2019/08/07/azure-functions-integration-testing-with-mountebank/
[post 2]: /2019/08/14/azure-functions-sre-on-azure-devops-the-first-cut/

[gh sample]: https://github.com/devkimchi/azure-functions-integration-testing

[az fncapp]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-39275-juyoo
[az fncapp openapi]: https://github.com/azure/azure-functions-openapi-extension
[az fncapp cli]: https://docs.microsoft.com/azure/azure-functions/functions-run-local?WT.mc_id=dotnet-39275-juyoo

[gha]: https://github.com/features/actions
