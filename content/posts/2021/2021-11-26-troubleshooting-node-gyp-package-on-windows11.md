---
title: "Troubleshooting node-gyp Package on Windows 11"
slug: troubleshooting-node-gyp-package-on-windows11
description: "In this post, I'm going to show how to sort out the infamous node-gyp issue on Windows 11 development environment."
date: "2021-11-26"
author: Justin-Yoo
tags:
- windows11
- nodejs
- node-gyp
- troubleshooting
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-00.png
fullscreen: true
---

It's a common practice using [node.js][node js] for front-end app development. In the Windows dev environment, the same exercise applies. If you use [Windows Subsystem for Linux (WSL)][wsl], you can make use of the Linux environment for it. But what if you want to keep your dev environment on Windows 11? One of the most infamous errors you might be able to see is related to the [`node-gyp`][node gyp] package. Throughout this post, I'm going to discuss how to fix the `node-gyp` error on Windows 11.


## `node-gyp` Package Dependencies ##

The [`node-gyp`][node gyp] package is a part of [npm][node npm], which is automatically installed while installing node.js. Therefore, you should be aware of your node.js and npm versions to get the issue sorted out. The latest LTS version of node.js at the time of this writing is `16.13.0` and npm of `8.1.0`.

![node.js and npm version][image-01]

In general, if there's a project on GitHub you are collaborating on, what could be the process to build an app?

1. Clone the repository to your local dev environment,
2. Run the `npm install` command to download node modules, and
3. Run the `npm run <command>` command to run the application

I know the steps identified above are exactly the same but similar. Depending on your case, you might use the `node-gyp` package, which you are likely to meet the errors below.


### Python ###

It doesn't matter whether you are developing Python apps or not. The `node-gyp` package has a direct dependency on [Python][python], and you have to install it beforehand.

You might see the error message like below while running the command, `npm install`.

> **NOTE**: The default version of `node-gyp` is `8.2.0`, which comes with node.js (LTS) `14.13.0`, as the screenshot says.

![Python not found error][image-02]

It's because Python is not installed on your machine. The solution is straightforward &ndash; install Python. Go to the [Python website][python], download the latest version and install it. The newest version of Python at the time of this writing is `3.10.0`.

![Python website][image-03]

Once installed, run the following command to check whether Python is correctly installed or not.

![Python version check][image-04]


### Visual Studio 2019 Build Tools ###

Run the `npm install` command again. Then, you'll see the following error.

![Visual Studio not found error][image-05]

The `node-gyp` package also depends on the C++ compiler, and your dev environment doesn't have it yet. Therefore, you can install [Visual Studio][vs] or [Standalone Build Tools][vs 2019 build tools]. In this section, let's use the [Standalone Build Tools][vs 2019 build tools] for now. While installing, choose the "Desktop development with C++" workload option.

![Visual Studio 2019 installation][image-06]

Since the download link currently offers the Visual Studio 2019 version at the time of this writing, you will be able to see the following screen after the installation.

![Visual Studio 2019 Build Tools installation][image-07]

The installed path is the following:

```text
C:\Program Files (x86)\Microsoft Visual Studio\2019\BuildTools
```

![Visual Studio 2019 Build Tools installation path][image-08]

Run the `npm install` command again. Then you'll get all the node modules installed with no error. However, if you still see the error like above, you should let npm know the version of Visual Studio like:

```powershell
npm config set msvs_version 2019
```

Then, everything will go alright with the `npm install` command.

> **NOTE**: Depending on the timing, the [Standalone Build Tools download link][vs 2019 build tools] offers you the latest version of Build Tools. Therefore, you might not download Visual Studio 2019 version but the later version. If you download the later version than Visual Studio 2019, try the following section.


### Visual Studio 2022 Build Tools ###

Recently, Visual Studio 2022 was released, and the Build Tool has also been upgraded to 2022. [This page][vs 2022 build tools] gives you the latest release of Build Tools. Choose the "Desktop development with C++" workload option, like before.

![Visual Studio 2022 installation][image-09]

The installed location looks like this:

```text
C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools
```

![Visual Studio 2022 Build Tools installation path][image-10]

But the `node-gyp` version that comes with node.js `14.13.0` doesn't support Visual Studio 2022. Therefore, you should update the npm version by running the following command:

```powershell
npm install -g npm
```

Once updated, the npm version is changed from `8.1.0` to `8.1.4`.

![npm version updated][image-11]

In addition to that, the `node-gyp` package also has been updated from `8.2.0` to `8.4.0`.

![node-gyp version updated][image-12]

Now, run the `npm install` command again, and it will properly install all the node modules. You can also override the Visual Studio version like:

```powershell
npm config set msvs_version 2022
```


### Visual Studio 2022 ###

In the previous two cases, you don't need to install Visual Studio but the Build Tools workload. This time, let's use [Visual Studio 2022][vs 2022 community] itself. Visual Studio 2022 has a different installed location because it's now running on the x64 mode natively.

```text
C:\Program Files\Microsoft Visual Studio\2022\Community
```

![Visual Studio 2022 installation path][image-13]

Your `node-gyp` version has already been updated to `8.4.0`. Hence, once you complete installing Visual Studio 2022, running the `npm install` command won't cause an issue. Then, of course, you can force the Visual Studio version like below:

```powershell
npm config set msvs_version 2022
```

> **NOTE**: In this post, I just use the Visual Studio Community Edition. But you can use Professional or Enterprise Edition.


## Long Path Issue ##

It's not related to the `node-gyp` package, but you will frequently see this issue while developing the node.js app on Windows. The [long path issue on Windows OS][win10 long path] is now resolved on Windows 11 through Local Group Policy Editor.

![Local Group Policy Editor][image-14]

Once open Local Group Policy Editor, navigate to "Local Computer Policy" ➡️ "Computer Configuration" ➡️ "Administrative Templates" ➡️ "System" ➡️ "File System" and open the "Enable Win32 long paths" item.

![Local Group Policy Editor][image-15]

This value is "Not Configured" as default. Enable it.

![Local Group Policy Editor updated][image-16]

Then, you don't have to suffer from the long path issue any longer.

---

So far, we've walked through the `node-gyp` issue while working with the node.js app on Windows 11. I hope this approach helps.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-04.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-05.png
[image-06]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-06.png
[image-07]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-07.png
[image-08]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-08.png
[image-09]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-09.png
[image-10]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-10.png
[image-11]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-11.png
[image-12]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-12.png
[image-13]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-13.png
[image-14]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-14.png
[image-15]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-15.png
[image-16]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/11/troubleshooting-node-gyp-package-on-windows11-16.png


[node js]: https://nodejs.org/en/
[node gyp]: https://github.com/nodejs/node-gyp
[node npm]: https://www.npmjs.com/

[wsl]: https://docs.microsoft.com/windows/wsl/about?WT.mc_id=dotnet-50103-juyoo&ocid=AID3035186

[python]: https://python.org/

[vs]: https://visualstudio.microsoft.com/?WT.mc_id=dotnet-50103-juyoo&ocid=AID3035186
[vs 2019 build tools]: https://visualstudio.microsoft.com/thank-you-downloading-visual-studio/?sku=BuildTools&WT.mc_id=dotnet-50103-juyoo&ocid=AID3035186
[vs 2022 build tools]: https://docs.microsoft.com/visualstudio/install/create-an-offline-installation-of-visual-studio?view=vs-2022&WT.mc_id=dotnet-50103-juyoo&ocid=AID3035186#step-1---download-the-visual-studio-bootstrapper
[vs 2022 community]: https://visualstudio.microsoft.com/thank-you-downloading-visual-studio/?sku=Community&rel=17&WT.mc_id=dotnet-50103-juyoo&ocid=AID3035186

[win10 long path]: https://docs.microsoft.com/windows/win32/fileio/maximum-file-path-limitation?tabs=powershell&WT.mc_id=dotnet-50103-juyoo&ocid=AID3035186
