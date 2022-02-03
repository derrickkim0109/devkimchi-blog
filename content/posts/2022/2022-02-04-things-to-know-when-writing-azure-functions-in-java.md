---
title: "Things to Know When Writing Azure Functions in Java"
slug: things-to-know-when-writing-azure-functions-in-java
description: "In this post, I'm discussing what to know when building and deploying an Azure Functions app in Java."
date: "2022-02-04"
author: Justin-Yoo
tags:
- azure
- azure-functions
- java
- intellij-idea
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2022/02/things-to-know-when-writing-azure-functions-in-java-00.png
fullscreen: true
---

[Azure Functions][az fncapp] doesn't only support C# but also supports many other languages, including Java, JavaScript, Python, PowerShell and so forth. Specifically, I'm going to discuss a few things to know when you're developing a Java-based Azure Functions app, especially on Mac.


## Local Development Environments ##

It's not that a big issue when using either [Visual Studio][vs] or [Visual Studio Code][vs code] for .NET function app development. In addition to that, it's not that a big issue when using [Visual Studio Code][vs code] for JavaScript-based function app development, either.

> Basic reference docs for Java devs to develop Azure Functions app, please refer to this [Azure Functions Java Developers' Guide][az fncapp java] page.

It's common that the majority of Java devs use [IntelliJ IDEA (IntelliJ)][intellij]. I believe that Java Function app development also uses IntelliJ. But if you're on Mac, you should know the following Mac-specific issues so that you don't get hiccups.


### Things to Know When Using IntelliJ ###

1. **IntelliJ Edition**:

    There's no dependency on the Spring framework for Azure Functions app development. Therefore, you don't need the Ultimate Edition for Function app development. Of course, if you want to utilise many other features that UE only supports, you should use UE. But basically, CE would be sufficient for Function app development.

    ![Comparison between IntelliJ UE and CE][image-01]

    It's also better to install the [Azure Tools for IntelliJ][intellij az toolkits] plug-in for the Function app development because it's bootstrapping and templating all the Function app related stuff.

    ![Azure Tools for IntelliJ][image-02]

    The doc page, [Building the First Java Function App with IntelliJ][az fncapp java intellij], provides essential information to build a Java function app and deploy it to Azure.

1. **Maven Build**:

    When you write an Azure Functions app with IntelliJ, it has the dependency on the [Maven build][maven]. So, if you want to integrate with the [Gradle build][gradle], follow this page, [Building an Azure Functions app with Gradle][az fncapp java gradle], and import it to IntelliJ.


### Things to Know When Using IntelliJ on Mac OS ###

Let's focus on the "Azure Functions app development". How IntelliJ works on Mac is different from doing on Windows in that context. The following two issues are only happening on Mac.

1. **Azure Functions Core Tools Loading**:

    If you install [Azure Functions Core Tools on Windows][az fncapp core tools win], open IntelliJ and run the Azure Functions app locally, it works with no issue.

    ![Run Azure Functions app via IntelliJ on Windows][image-03]

    However, if you install [Azure Functions Core Tools on Mac][az fncapp core tools mac], open IntelliJ from Finder or Dock, and run the Azure Functions app locally, it throws the error like below &ndash; it doesn't recognise Azure Functions Core Tools.

    ![Run Azure Functions app via IntelliJ on Mac via Finder][image-04]

    By the way, you can open IntelliJ from Terminal by typing the `idea .` command.

    ![Open IntelliJ from Terminal][image-05]

    With this IntelliJ instance, run the Azure Functions app locally. Then, it recognises [Azure Functions Core Tools][az fncapp core tools mac] and runs the app with no issue.

    ![Run Azure Functions app via IntelliJ on Mac][image-06]

    Why does this discrepancy occur? It's because the loaded environment variables are different between "Finder-launched application" and "Shell-launched application". Due to this difference, JVM is loaded differently. This issue [has already been addressed][intellij az toolkits issue 1].

    When you see the `$PATH` variable from the "Finder-launched IntelliJ", it looks like this:

    ![$PATH value from Finder-launched IntelliJ][image-07]

    Let's check the same `$PATH` variable from the "Shell-launched IntelliJ". Can you see the difference?

    ![$PATH value from Shell-launched IntelliJ][image-08]

    It's not fixed yet. It's not even sure whether the issue should go to the plugin or IntelliJ. Therefore, the only workaround, for now, is that you should open the IntelliJ instance from the command line.

1. **Azure Functions Port Lockout**:

    You run the Azure Functions App within IntelliJ, stop running the app, and re-run the app. Then, you will see the following error &ndash; the port 7071 is still being used.

    ![Port is still being used][image-09]

    Azure Functions uses two processes &ndash; main process and sub-process. The Azure Functions runtime runs on the main process, and the language worker runs on the sub-process. In the shell environment or [Visual Studio Code][vs code], all the child processes are also terminated when the main process is terminated. On the other hand, IntelliJ only kills the main process, not the child process, which occupies the existing port.

    In this case, you have to manually find the process holding the port number and kill it. Here are the sample bash commands.

    https://gist.github.com/justinyoo/a41dc3bb8c19141f3844f214593956c8?file=01-kill-process.sh

    Alternatively, you can download the following shell script and run it when necessary.

    https://gist.github.com/justinyoo/045201b2c4280c818ca872af6284c720?file=kill-process.sh

    As I've also [piled an issue on the plugin repository][intellij az toolkits issue 2], I'll update it here how it's going.


## Azure Deployment Environment ##

According to the doc, Azure Functions supports both [Java 8 and 11][az fncapp java version]. Therefore, you need to configure the `pom.xml` file with the value of 1.8 or 11 for the `java.version` property and the value of 8 or 11 for the `javaVersion` property (line #6-7, 18-19, 32).

https://gist.github.com/justinyoo/a41dc3bb8c19141f3844f214593956c8?file=02-pom.xml&highlights=6-7,18-19,32


But, when you actually deploy the app with Java 11, it's deployed perfectly but not accessible. The `pom.xml` only accepts 1.8/8 for Azure deployment, which I suspect it's a bug at the time of this writing. I'm sure it's a known issue. In the meantime, you should target Java 1.8/8 for deployment.

---

So far, we've discussed how to write an Azure Functions app in Java, what possible issues you should know if you're developing it on Mac, and what you should know when you deploy the app to Azure. While developing the app, I guess you will face other issues, but these issues and workarounds identified in this post will be the bare minimum points you should know.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/02/things-to-know-when-writing-azure-functions-in-java-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/02/things-to-know-when-writing-azure-functions-in-java-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/02/things-to-know-when-writing-azure-functions-in-java-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/02/things-to-know-when-writing-azure-functions-in-java-04.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/02/things-to-know-when-writing-azure-functions-in-java-05.png
[image-06]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/02/things-to-know-when-writing-azure-functions-in-java-06.png
[image-07]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/02/things-to-know-when-writing-azure-functions-in-java-07.png
[image-08]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/02/things-to-know-when-writing-azure-functions-in-java-08.png
[image-09]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/02/things-to-know-when-writing-azure-functions-in-java-09.png


[gh sample]: https://github.com/fusiondevkr/fusiondevkr

[vs]: https://visualstudio.microsoft.com/vs/?WT.mc_id=dotnet-56308-juyoo
[vs code]: https://code.visualstudio.com/?WT.mc_id=dotnet-56308-juyoo

[intellij]: https://www.jetbrains.com/idea/
[intellij az toolkits]: https://plugins.jetbrains.com/plugin/8053-azure-toolkit-for-intellij
[intellij az toolkits issue 1]: https://github.com/microsoft/azure-tools-for-java/issues/6243#issuecomment-1012500771
[intellij az toolkits issue 2]: https://github.com/microsoft/azure-tools-for-java/issues/6374

[maven]: https://maven.apache.org/
[gradle]: https://gradle.org/

[az fncapp]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-56308-juyoo
[az fncapp java]: https://docs.microsoft.com/azure/azure-functions/functions-reference-java?tabs=bash%2Cconsumption&WT.mc_id=dotnet-56308-juyoo
[az fncapp java version]: https://docs.microsoft.com/azure/azure-functions/functions-reference-java?tabs=bash%2Cconsumption&WT.mc_id=dotnet-56308-juyoo#java-versions
[az fncapp java intellij]: https://docs.microsoft.com/azure/azure-functions/functions-create-maven-intellij?WT.mc_id=dotnet-56308-juyoo
[az fncapp java gradle]: https://docs.microsoft.com/azure/azure-functions/functions-create-first-java-gradle?WT.mc_id=dotnet-56308-juyoo
[az fncapp core tools win]: https://docs.microsoft.com/azure/azure-functions/functions-run-local?tabs=v4%2Cwindows%2Cjava%2Cportal%2Cbash%2Ckeda&WT.mc_id=dotnet-56308-juyoo#install-the-azure-functions-core-tools
[az fncapp core tools mac]: https://docs.microsoft.com/azure/azure-functions/functions-run-local?tabs=v4%2Cmacos%2Cjava%2Cportal%2Cbash&WT.mc_id=dotnet-56308-juyoo#install-the-azure-functions-core-tools
