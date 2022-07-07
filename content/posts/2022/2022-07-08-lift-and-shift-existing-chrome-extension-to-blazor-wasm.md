---
title: "Lift & Shift Existing Chrome Extension to Blazor WebAssembly"
slug: lift-and-shift-existing-chrome-extension-to-blazor-wasm
description: "Throughout this post, I'm going to walk through how to migrate the existing Chrome extension with minimal code changes."
date: "2022-07-08"
author: Justin-Yoo
tags:
- dotnet
- blazor-wasm
- chrome-extension
- migration
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-00.png
fullscreen: true
---

A [Chrome extension][chrome extension] is a tiny app used for Chromium-based web browsers for many purposes. It's like a web application serving static contents, which doesn't need a web server. If this is the case, what if we can build the Chrome extension using [Blazor WebAssembly][blazor wasm]? Throughout this post, I'm going to migrate the existing JavaScript-based Chrome extension to Blazor WebAssembly with minimal code changes.

> You can download the sample Chrome extension from [this GitHub repository][gh sample].


## Chrome Extension &ndash; JavaScript-based ##

There's a [Chrome extension app][gh sample v2 original] based on the [Manifest v2][chrome extension v2]. Load it to your [Edge browser][edge], and it will look like the following image.

![Chrome Extension on Microsoft Edge][image-01]

You can change the background colour if you run it on your web browser.

![Chrome Extension to change background colour #1][image-02]
![Chrome Extension that has changed the background colour #1][image-03]

Because it's written in JavaScript, let's migrate this app to Blazor WebAssembly.

> Due to the policy change, you cannot publish any Chrome extension based on the manifest v2 to Web Store from January 17th, 2022. I'm going to discuss the manifest v3 extension in the following posts.


## Chrome Extension &ndash; Blazor WebAssembly-based ##

Create a Blazor WebAssembly project. In the newly created project, open the `index.html` file, and you will see the `div` tag having the `id` attribute of `app`.

![Visual Studio][image-04]

Basically, Blazor WebAssembly handles the [virtual DOM][vdom] within the `div` tag with the `app` ID. Therefore, your Chrome extension should accommodate it. In the coming posts, I'll talk about how you manage the virtual DOM in a Blazor WebAssembply app.

There are several differences between the Chrome extension and the Blazor WebAssembly app. One of the differences is routing. Because Blazor WebAssembly relies on the routing with the virtual DOM, you only need one physical file, `index.html`.

Delete both `Counter.razor` and `FetchData.razor`, and add `Popup.razor` page.

```razor
@* Popup.razor *@

@page "/popup.html"

<PageTitle>Pop Up</PageTitle>

<h1>Pop Up - Chrome Extension with Blazor WASM</h1>

<button id="changeColor"></button>
```

Then, add `Options.razor` page.

```razor
@* Options.razor *@

@page "/options.html"

<PageTitle>Options</PageTitle>

<h1>Options - Chrome Extension with Blazor WASM</h1>

<div id="buttonDiv">
</div>
<div>
    <p>Choose a different background color!</p>
</div>
```

As you can see, each Razor page has the routing value of `popup.html` and `options.html`, respectively.

Once all is done, publish your Blazor WebAssembly app:

```powershell
dotnet publish ./src/ChromeExtensionV2 -c Release -o published
```

You can confirm the published artifact:

![Chrome Extension published #1][image-05]

It's time to register this migrated extension. However, you can't register it because of the error message below. The error message says that you can't use any directory name or filename starting with an underscore (`_`). Unfortunately, the Blazor WebAssembly app generates the `_framework` directory by default. Therefore, you can't directly use the published artifact.

![Chrome Extension registration error #1][image-06]

It would be best if you changed all the underscored directories and filenames. In addition to that, it would be best if you change all the contents referencing underscored files or directories. You can easily do this with a PowerShell script or a bash shell script. I'm going to use the PowerShell script for this post.

```powershell
# Run-PostBuild.ps1

function Update-FileContent {
    param (
        [string] $Filename,
        [string] $Value1,
        [string] $Value2
    )

    $content = Get-Content -Path $Filename -Raw
    $updated = $content -replace $Value1, $Value2
    Set-Content -Path $Filename -Value $updated -Force
}

Update-FileContent `
    -Filename "./published/wwwroot/_framework/blazor.webassembly.js" `
    -Value1 "_framework" `
    -Value2 "framework"

Update-FileContent `
    -Filename "./published/wwwroot/index.html" `
    -Value1 "_framework" `
    -Value2 "framework"

mv ./published/wwwroot/_framework ./published/wwwroot/framework
```

After running the PowerShell script, you'll be able to see the directory name changed.

![Chrome Extension published #2][image-07]

Register the extension again. This time, another error occurs that no `options.html` file exists. In other words, you physically need the `options.html` file.

![Chrome Extension registration error #2][image-08]

According to this error, any Chrome extension must have `popup.html` and/or `options.html` files. Both Razor files are not enough for page rendering. By the way, both `popup.html` and `options.html` files are fundamentally the same as the `index.html` that access the `div` tag having the `app` ID to handle the virtual DOM. So then, instead of manually creating both files, let's duplicate the existing `index.html` file by adding those commands to the PowerShell script.

```powershell
# Run-PostBuild.ps1
...
cp ./published/wwwroot/index.html ./published/wwwroot/popup.html
cp ./published/wwwroot/index.html ./published/wwwroot/options.html
```

Rerun the script. Both `popup.html` and `options.html` are duplicated from `index.html`.

![Chrome Extension published #3][image-09]

Register the extension again. No error has occurred.

![Chrome Extension registered #1][image-10]

However, no error occurring during the registration process doesn't mean that the extension actually works. Click the "Inspect pop-up window" menu like the screenshot below.

![Chrome Extension inspect pop-up window][image-11]

Then, you'll see the developer tools window that says there are errors. It's because of the [Content Security Policy][mdn content security policy].

![Chrome Extension pop-up error #1][image-12]

To sort out this error, the `manifest.json` file needs the `content_security_policy` attribute.

```json
{
  "manifest_version": 2,
  "version": "1.0",
  "name": "Getting Started Example (Blazor WASM)",
  "description": "Build an Extension!",

  ...

  "content_security_policy": "script-src 'self' 'unsafe-eval' 'wasm-unsafe-eval' 'sha256-v8v3RKRPmN4odZ1CWM5gw80QKPCCWMcpNeOmimNL2AA='; object-src 'self'",

  ...
}
```

Here are some details for the `content_security_policy` attribute:

* `unsafe-eval`: For the `Function()` object, and `setTimeout()` and `setInterval()` functions.
* `wasm-unsafe-eval`: For the web assembly binary files.
* `sha256-v8v3RKRPmN4odZ1CWM5gw80QKPCCWMcpNeOmimNL2AA=`: For the Blazor WebAssembly libraries.

After updating the `manifest.json` file, register the extension again. Everything looks OK.

![Chrome Extension registered #2][image-13]

However, we still don't see the button to change the background colour. It means that the extension still doesn't work as expected. It's because both `popup.html` and `options.html` don't have corresponding JavaScript files.

![Chrome Extension popup.html #1][image-14]

Both files are automatically generated from `index.html` through the PowerShell script. Therefore, add the JavaScript reference of `js/main.js` to `index.html`. Also, create an empty `main.js` file under the `js` directory.

```html
<!DOCTYPE html>
<html lang="en">
...
<body>
    <div id="app">Loading...</div>
    ...
    <script src="_framework/blazor.webassembly.js"></script>
    <!-- ⬇️⬇️⬇️ Add this line ⬇️⬇️⬇️ -->
    <script src="js/main.js"></script>
    <!-- ⬆️⬆️⬆️ Add this line ⬆️⬆️⬆️ -->
</body>
</html>
```

Then, update the PowerShell script by adding the following lines so that both `popup.html` and `options.html` have the link to `popup.js` and `options.js`, respectively.

```powershell
# Run-PostBuild.ps1
...
Update-FileContent `
    -Filename "./published/wwwroot/popup.html" `
    -Value1 "js/main.js" `
    -Value2 "js/popup.js"

Update-FileContent `
    -Filename "./published/wwwroot/options.html" `
    -Value1 "js/main.js" `
    -Value2 "js/options.js"
```

Re-register the extension and see how it's going. But another error has occurred this time. Both JavaScript files worked perfectly with no issue but not in the Blazor WebAssembly. Why is that?

![Chrome Extension pop-up error #2][image-15]

The `blazor.webassembly.js` file knows the answer. This JavaScript file initiates the Blazor WebAssembly app, followed by loading either `popup.js` or `options.js`. Unfortunately, those files are loaded even before the Blazor app is initiated, which causes the error above.

Therefore, to figure out this problem, you need to update the JavaScript reference section like below. [JS interop for Blazor WebAssembly][blazor wasm jsinterop] has more information if you want to know more.

```html
<!DOCTYPE html>
<html lang="en">
...
<body>
    <div id="app">Loading...</div>
    ...
    <!-- Add the 'autostart' attribute and set its value to 'false' -->
    <script src="_framework/blazor.webassembly.js" autostart="false"></script>
    <!-- ⬇️⬇️⬇️ Add these lines ⬇️⬇️⬇️ -->
    <script>
        Blazor.start().then(function () {
            var customScript = document.createElement('script');
            customScript.setAttribute('src', 'js/main.js');
            document.head.appendChild(customScript);
        });
    </script>
    <!-- ⬆️⬆️⬆️ Add these lines ⬆️⬆️⬆️ -->
</body>

</html>
```

Build the app and register the extension again. There's another Content Security Policy-related error.

![Chrome Extension pop-up error #3][image-16]

Add the hash value mentioned in error to `manifest.json` to fix this error.

```json
{
  "manifest_version": 2,
  "version": "1.0",
  "name": "Getting Started Example (Blazor WASM)",
  "description": "Build an Extension!",

  ...

  "content_security_policy": "script-src 'self' 'unsafe-eval' 'wasm-unsafe-eval' 'sha256-v8v3RKRPmN4odZ1CWM5gw80QKPCCWMcpNeOmimNL2AA=' 'sha256-DnTH4SKCYpHBGu1OxDOqoYLsvmZTiYIWJVQ1Ava7Kig=' 'sha256-b9roSuk6Pa7l0Hl/LWXGQlupw8fMc6ME2+82/N3qM0Q='; object-src 'self'",

  ...
}
```

What are those hash values?

* `sha256-DnTH4SKCYpHBGu1OxDOqoYLsvmZTiYIWJVQ1Ava7Kig=`: It points to `popup.js`.
* `sha256-b9roSuk6Pa7l0Hl/LWXGQlupw8fMc6ME2+82/N3qM0Q=`: It points to `options.js`.

Once added, register the extension again. Now it's working as expected!

![Chrome Extension to change background colour #2][image-17]
![Chrome Extension that has changed the background colour #2][image-18]

Your existing JavaScript-based Chrome extension has now successfully been migrated to Blazor WebAssembly! If you want to see the complete example, go to [this page][gh sample v2 blazor].

---

So far, we've migrated the JavaScript-based Chrome extension app to [Blazor WebAssembly][blazor wasm] with minimal code changes. Both extensions are almost identical, except for a couple of points. In other words, your extension can easily be migrated using the "lift & shift" approach. However, it doesn't entirely make use of the powerful [JS interop][blazor wasm jsinterop] feature yet. In the next post, let's explore how we can use the JS interop feature more for this migration.


## Do you want to know more about Blazor? ##

Here are some tutorials for you.

* [Blazor][blazor]
* [Blazor Tutorial][blazor tutorial]
* [Blazor Learn][blazor learn]


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-02-ko.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-03-ko.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-04.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-05.png
[image-06]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-06.png
[image-07]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-07.png
[image-08]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-08.png
[image-09]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-09.png
[image-10]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-10.png
[image-11]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-11-ko.png
[image-12]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-12.png
[image-13]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-13-ko.png
[image-14]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-14.png
[image-15]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-15-ko.png
[image-16]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-16-ko.png
[image-17]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-17-ko.png
[image-18]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-18-ko.png


[gh sample]: https://github.com/devkimchi/blazor-wasm-chrome-extension/tree/the-migration
[gh sample v2 original]: https://github.com/devkimchi/blazor-wasm-chrome-extension/tree/the-migration/src/chrome-extension-v2
[gh sample v2 blazor]: https://github.com/devkimchi/blazor-wasm-chrome-extension/tree/the-migration/src/ChromeExtensionV2

[chrome extension]: https://developer.chrome.com/docs/extensions/
[chrome extension v2]: https://developer.chrome.com/docs/extensions/mv2/getstarted/

[blazor]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor?WT.mc_id=dotnet-70466-juyoo
[blazor tutorial]: https://dotnet.microsoft.com/learn/aspnet/blazor-tutorial/intro?WT.mc_id=dotnet-70466-juyoo
[blazor learn]: https://docs.microsoft.com/learn/paths/build-web-apps-with-blazor/?WT.mc_id=dotnet-70466-juyoo
[blazor wasm]: https://docs.microsoft.com/aspnet/core/blazor/host-and-deploy/webassembly?view=aspnetcore-6.0&WT.mc_id=dotnet-70466-juyoo
[blazor wasm jsinterop]: https://docs.microsoft.com/aspnet/core/blazor/javascript-interoperability/?view=aspnetcore-6.0&WT.mc_id=dotnet-70466-juyoo

[edge]: https://www.microsoft.com/edge?WC.mc_id=dotnet-70466-juyoo

[vdom]: https://en.wikipedia.org/wiki/Virtual_DOM

[mdn content security policy]: https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/manifest.json/content_security_policy
