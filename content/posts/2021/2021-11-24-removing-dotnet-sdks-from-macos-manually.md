---
title: "Removing .NET SDKs from MacOS Manually"
slug: removing-dotnet-sdks-from-macos-manually
description: "In this post, I'm going to walk through how to manually remove a specific version of .NET SDK and runtime from MacOS."
date: "2021-11-24"
author: Justin-Yoo
tags:
- dotnet
- dotnet-sdk
- macos
- installation
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/removing-dotnet-sdks-from-macos-manually-00.png
fullscreen: true
---

.NET SDKs including .NET Core, .NET 5 and .NET 6 support cross-platform. For some reason, you might feel like removing the latest preview version of SDK or a certain version of SDK. You can then use many different methods to delete the SDK and runtime. In this post, I'm going to discuss the **manual** way of removing a specific version of the .NET SDK and runtime from MacOS.


## Installation Paths for .NET SDKs & Runtimes ##

In your terminal in Mac OS, type the command, `dotnet --list-sdks`, and you will see the whole list of .NET SDKs you've installed so far, including the full path.

![Installed .NET SDKs][image-01]

This time, run the command, `dotnet --list-runtimes`, to get the whole list of the installed .NET runtimes.

![Installed .NET Runtimes][image-02]

Did you find some differences between SDKs and runtimes, other than the paths? That's right. Although you say .NET 6, the SDK and the runtime versions are slightly different. The same goes to .NET 5, .NET Core 3.1 and .NET Core 2.1, etc. Therefore, you need to be careful when the time comes to delete both SDK and runtime manually.

> **NOTE**: If you want to know more about the .NET SDK and runtime versions, visit this page, [How to check that .NET is already installed][dotnet check].


## Deleting .NET SDK and Runtime ##

First of all, you can use the [.NET Uninstall Tool][dotnet uninstall tool] to automatically delete the SDK and runtime you want. But as the doc says, sometimes it's not perfect deleting. Therefore, at the end of the day, you might want to delete each directory manually.

As mentioned above, you should be aware that both the SDK and runtime versions of the same release are different. For example, at the time of this writing, the latest release of .NET 6 has the SDK version of `6.0.100` and runtime version of `6.0.0`. On the other hand, in case of .NET 5, the SDK version is `5.0.403`, and the runtime version is `5.0.12`. Therefore, to manually delete all the related directories, it is best to refer to this page, [How to remove the .NET Runtime and SDK][dotnet uninstall manual]. But let's dive a little deep here.


### Manually Deleting .NET SDK ###

Assuming you're deleting .NET 6 SDK. The latest version of the .NET 6 SDK is `6.0.100` at the time of this writing. As there are only two relevant directories to SDK, run the following bash command:

https://gist.github.com/justinyoo/68c11fa00480fa15d38f7288815c6ba5?file=01-remove-sdk.sh

The .NET SDKs have been completely removed.


### Manually Deleting .NET Runtime ###

This time, let's delete the .NET 6 runtime directories. Then, run the following bash command. Beware that number of directories varies, depending on the version of the runtime. But, since this is the whole list of the runtime directories, there's no harm to run the command anyway.

https://gist.github.com/justinyoo/68c11fa00480fa15d38f7288815c6ba5?file=02-remove-runtime.sh

The specified .NET runtime directories have all been removed.


## Manually Deleting Both SDK and Runtime at Once via PowerShell ##

If you're familiar with bash shell scripting, you can create a more sophisticated one to do the job. Instead, let's use [PowerShell][pwsh] to delete both specified SDK and runtime at once. PowerShell also supports cross-platform, meaning you can use PowerShell on your MacOS. Here's the sample script.

https://gist.github.com/justinyoo/68c11fa00480fa15d38f7288815c6ba5?file=03-remove-both.ps1

By running this PowerShell script above, you can delete both the SDK and runtime of a specific version at once.


## Reinstalling .NET SDK & Runtime of Previous Versions ##

I guess this is the most crucial part. From time to time, after the manual SDK/runtime deleting is done, the remaining .NET SDKs and runtimes won't work. If it happens to you, then you should reinstall the existing .NET SDKs and runtimes. For example, if you delete .NET 6 SDK and runtime, then suddenly .NET Core 2.1 and 3.1, and .NET 5 start complaining, you should reinstall the latest release of each .NET SDKs and runtimes. To get the old release, visit this [download][dotnet download] page, download the corresponding version of your choice and install them.

![.NET SDK Download Page][image-03]

---

So far, I've walked through how to delete a specific .NET SDK and runtime from MacOS manually. In most cases, you won't need this approach, but I hope this walkthrough will be helpful if you do.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/removing-dotnet-sdks-from-macos-manually-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/removing-dotnet-sdks-from-macos-manually-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/removing-dotnet-sdks-from-macos-manually-03.png


[dotnet check]: https://docs.microsoft.com/dotnet/core/install/how-to-detect-installed-versions?pivots=os-macos&WT.mc_id=dotnet-50035-juyoo&ocid=AID3035186
[dotnet download]: https://dotnet.microsoft.com/download/dotnet?WT.mc_id=dotnet-50035-juyoo&ocid=AID3035186
[dotnet uninstall tool]: https://docs.microsoft.com/dotnet/core/additional-tools/uninstall-tool?tabs=macos&WT.mc_id=dotnet-50035-juyoo&ocid=AID3035186
[dotnet uninstall manual]: https://docs.microsoft.com/dotnet/core/install/remove-runtime-sdk-versions?pivots=os-macos&WT.mc_id=dotnet-50035-juyoo&ocid=AID3035186

[pwsh]: https://docs.microsoft.com/powershell/scripting/overview?view=powershell-7.2&WT.mc_id=dotnet-50035-juyoo&ocid=AID3035186
