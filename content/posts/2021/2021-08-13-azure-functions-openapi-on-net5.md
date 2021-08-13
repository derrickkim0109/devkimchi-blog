---
title: "Azure Functions OpenAPI Extensions on .NET 5 Isolated Worker Environment"
slug: azure-functions-openapi-on-net5
description: "In this post, I'm going to show how to run the OpenAPI extension for Azure Functions on the .NET 5 isolated worker environment."
date: "2021-08-13"
author: Justin-Yoo
tags:
- azure
- azure-functions
- openapi
- swagger
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-00.png
fullscreen: true
---

In May this year at [//Build][build2021], [Azure Functions][az fncapp] [OpenAPI support feature (preview)][azfunc openapi] was [officially announced][build2021 openapi]. At that time, it supported up to the v3 runtime &ndash; .NET Core 3.1 version. Recently, it has released [.NET 5 isolated worker supporting package][azfunc openapi nuget] as a preview. Throughout this post, I'm going to review how to use it and deploy it to Azure.

> NOTE: You can find the sample code used in this post at [this GitHub repository][gh sample].


## Creating Azure Functions App in .NET 5 ##

Let's use [Visual Studio][vs] for this exercise. While creating the app, use the ".NET 5 (Isolated)" runtime and "Http trigger".

![Choose .NET 5 (Isolated) then Http trigger][image-01]

Then you'll find out the HTTP endpoint with default codes. Now, select the NuGet Package Manager menu at Solution Explorer.

![Select NuGet Package Manager menu][image-02]

In the NuGet Package Manager screen, tick the "Include prerelease" checkbox and search up the `Microsoft.Azure.Functions.Worker.Extensions.OpenApi` package. As of this writing, the NuGet packager version is `v0.8.1-preview`.

![Search up the OpenAPI extension in the NuGet Package Manager screen][image-03]

OpenAPI extension has now been installed.


## Configuring HostBuilder ##

After installing the OpenAPI extension, let's configure the `HostBuilder`. First, open the `Program.cs` file and remove the existing `ConfigureFunctionsWorkerDefaults()` method. It's because this method uses `System.Text.Json` by default, which we're not going to use.

https://gist.github.com/justinyoo/32474da3f9e512b6ab4e34192fbbb4b8?file=00-program.cs

Then, add both `ConfigureFunctionsWorkerDefaults(worker => worker.UseNewtonsoftJson())` and `ConfigureOpenApi()` method in this order. The first method explicitly declares to use the `Newtonsoft.Json` package and the next one imports the additional OpenAPI related endpoints.

https://gist.github.com/justinyoo/32474da3f9e512b6ab4e34192fbbb4b8?file=01-program.cs

> **NOTE**: Currently, using `System.Text.Json` doesn't guarantee whether the app works appropriately or not. Therefore, `Newtonsoft.Json` is highly recommended.

Now, the configuration is over. Let's move on.


## Adding OpenAPI Decorators ##

Add OpenAPI related decorators like below. It's precisely the same exercise as the existing approach, so I'm not going too deep.

https://gist.github.com/justinyoo/32474da3f9e512b6ab4e34192fbbb4b8?file=03-run.cs

Once you complete adding the decorators, you're all done! Let's run the app.


## Running Swagger UI ##

Run the Function app by entering the `F5` key or clicking the debug button on [Visual Studio][vs].

![Run Azure Function app with the debug mode][image-04]

You'll see the OpenAPI related endpoints added in the console.

![OpenAPI related endpoints added][image-05]

Run the `http://localhost:7071/api/swagger/ui` endpoint on your web browser, and you'll see the Swagger UI page.

![Swagger UI page][image-06]

Your Azure Function app with OpenAPI capability has now been implemented correctly.


## Deploying Azure Function App &ndash; Windows ##

You confirmed that your Azure Function app is working fine. It's time for deployment. First of all, click the "Publish" menu at Solution Explorer.

![Azure Functions publish menu][image-07]

Choose "Azure", followed by "Azure Functions App (Windows)".

![Choose deployment method for Windows][image-08]

You can use the existing Function app instance or create a new one by clicking the ➕ button. This time, let's use the current instance.

![Choose existing instance][image-09]

Once deployment is done, open the Azure Functions URL on your web browser, and you'll see the Swagger UI page.

![Swagger UI on Azure][image-10]


## Deploying Azure Function App &ndash; Linux ##

This time, let's deploy the same app to the Linux instance. In addition to that, let's use GitHub Actions. In order to do so, you must upload this app to a GitHub repository. So, move to the "Git Changes" pane and create a Git repository.

![Git Changes pane][image-11]

If you've already logged in to GitHub within [Visual Studio][vs], you'll be able to create a repository and push the codes.

![Create GitHub repository][image-12]

Once pushed all codes, visit GitHub to check your repository whether all codes have actually been uploaded.

![Complete creating GitHub repository][image-13]

Let's go back to the publish screen and click the "➕ New" button to create a new publish profile.

![New publish profile][image-14]

Then a similar pop-up appears to before. This time let's use the "Azure Function App (Linux)" menu.

![Choose the deployment method for Linux][image-15]

As mentioned before, you can use an existing instance or create a new one. Let's use the existing one.

![Choose existing instance][image-16]

In the previous deployment exercise, we haven't got a GitHub repository. Therefore, we had to use the local deployment method. But this time, we've got the GitHub repository, meaning we've got choices. Therefore, instead of choosing the same deployment method, let's choose GitHub Actions for this time.

![Choose GitHub Actions][image-17]

GitHub Actions workflow has now been automatically generated. But this needs a new commit.

![GitHub Actions commit & push][image-18]

Move to the "Git Changes" pane, enter the commit message like below, click the "Commit All" button, and push the change.

![Commit & push the GitHub Actions workflow][image-19]

When you actually visit your GitHub repository, your GitHub Actions workflow runs the build and deployment.

![Deploying Function app with GitHub Actions][image-20]

Once the deployment is over, open a new web browser, visit the Azure Functions app URL, and find out the Swagger UI page is correctly rendered.

![Swagger UI on Azure][image-21]

---

So far, we've walked through how to create an [OpenAPI enabled][azfunc openapi] [Azure Functions][az fncapp] app, running on .NET 5 isolated worker environment, and deploy it to Azure without having to leave [Visual Studio][vs]. I'm guessing that it will run OK on .NET 6, theoretically. If you're curious, please deploy it and let me know!


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-04.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-05.png
[image-06]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-06.png
[image-07]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-07.png
[image-08]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-08.png
[image-09]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-09.png
[image-10]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-10.png
[image-11]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-11.png
[image-12]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-12.png
[image-13]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-13.png
[image-14]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-14.png
[image-15]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-15.png
[image-16]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-16.png
[image-17]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-17.png
[image-18]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-18.png
[image-19]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-19.png
[image-20]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-20.png
[image-21]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/08/azure-functions-openapi-on-net5-21.png


[gh sample]: https://github.com/justinyoo/azfunc-openapi-dotnet

[build2021]: https://mybuild.microsoft.com/home?WT.mc_id=dotnet-38365-juyoo
[build2021 openapi]: https://techcommunity.microsoft.com/t5/apps-on-azure/create-and-publish-openapi-enabled-azure-functions-with-visual/ba-p/2381067?WT.mc_id=dotnet-38365-juyoo

[azfunc openapi]: https://github.com/Azure/azure-functions-openapi-extension
[azfunc openapi nuget]: https://github.com/Azure/azure-functions-openapi-extension/releases/tag/v0.8.1-preview

[az fncapp]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-38365-juyoo

[vs]: https://visualstudio.microsoft.com/?WT.mc_id=dotnet-38365-juyoo
