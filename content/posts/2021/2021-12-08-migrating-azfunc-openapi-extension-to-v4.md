---
title: "Migrating Azure Functions OpenAPI Extension to V4"
slug: migrating-azfunc-openapi-extension-to-v4
description: "In this post, I'm going to show how to migrate the existing .NET Core 2.1/3.1 or .NET 5 function app to .NET 6, without any codebase update."
date: "2021-12-08"
author: Justin-Yoo
tags:
- azure
- azure-functions
- openapi-extension
- migration
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/12/migrating-azfunc-openapi-extension-to-v4-00.png
fullscreen: true
---

In early November, 2021 [Azure Functions][az fncapp] announced [its GA for .NET 6 support and V4 runtime][az fncapp ga]. At the same time, [Azure Functions OpenAPI Extension also announced its GA][az fncapp ga openapi], which supports all .NET Core 2.1 LTS, 3.1 LTS, .NET 5 and .NET 6. In this post, I'm going to discuss what codebase needs to be updated to support .NET 6 and the V4 runtime.


## OpenAPI Extension Update ##

This GA package has now removed the `-preview` tag. Therefore, if you want to keep the existing runtime, simply change the version to `1.0.0` (line #5). Here's the `.csproj` file that you need to change if your app uses either .NET Core 2.1 or 3.1.

https://gist.github.com/justinyoo/8fbd8f285c55a351be2cbf08d8119ce6?file=01-fncapp-31-package-reference.xml&highlights=5

If you've been using both .NET 5 and out-of-proc worker, also change the version to `1.0.0` (line #5).

https://gist.github.com/justinyoo/8fbd8f285c55a351be2cbf08d8119ce6?file=02-fncapp-50-package-reference.xml&highlights=5

Your existing app doesn't require any code change by updating the package version and should just work.

But, it's a different story when you upgrade the function runtime from V3 to V4. In that case, you also need to change another part of the `.csproj` file. In addition to that, You must upgrade your [Azure Functions Core Tools][az fncapp core tools] to V4 for your local development.


## Migrating .NET Core 3.1 to .NET 6 ##

Open the `.csproj` file and find the `TargetFramework` element. Then change its value from `netcoreapp3.1` to `net6.0`. After that, look for the `AzureFunctionsVersion` element and change the value from `v3` to `v4` (*line #4-5, 8-9*).

https://gist.github.com/justinyoo/8fbd8f285c55a351be2cbf08d8119ce6?file=03-fncapp-31-target-framework.xml&highlights=4-5,8-9

Once it's done, rebuild the entire solution and run the app. It should just work fine.

> **NOTE**: If you use .NET Core 2.1, then change the value of the `TargetFramework` element from `netcoreapp2.1` to `net6.0`.


## Migrating .NET 5 to .NET 6 ##

The same rules apply. Open the `.csproj` file and find the `TargetFramework` element. Then change its value from `net5.0` to `net6.0`. After that, look for the `AzureFunctionsVersion` element and change the value from `v3` to `v4` (*line #4-5, 8-9*).

https://gist.github.com/justinyoo/8fbd8f285c55a351be2cbf08d8119ce6?file=04-fncapp-50-target-framework.xml&highlights=4-5,8-9

Once it's done, rebuild the entire solution and run the app. It should just work fine.

---

Did I change any codebase? No. All you need to do is to change the extension package version, target framework version and runtime version. Then, it should just work. Unless one of the packages you're referencing has a dependency on .NET Core 2.1/3.1 or .NET 5, you'll be able to enjoy the latest version of the .NET and OpenAPI extension.


[az fncapp]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-44909-juyoo
[az fncapp core tools]: https://docs.microsoft.com/azure/azure-functions/functions-run-local?tabs=v4%2Cwindows%2Ccsharp%2Cportal%2Cbash%2Ckeda&WT.mc_id=dotnet-44909-juyoo
[az fncapp ga]: https://techcommunity.microsoft.com/t5/apps-on-azure-blog/azure-functions-4-0-and-net-6-support-are-now-generally/ba-p/2933245?WT.mc_id=dotnet-44909-juyoo
[az fncapp ga openapi]: https://techcommunity.microsoft.com/t5/apps-on-azure-blog/general-availability-of-azure-functions-openapi-extension/ba-p/2931231?WT.mc_id=dotnet-44909-juyoo

