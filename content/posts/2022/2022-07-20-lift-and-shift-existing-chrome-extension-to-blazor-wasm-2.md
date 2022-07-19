---
title: "Lift & Shift Existing Chrome Extension to Blazor WebAssembly #2"
slug: lift-and-shift-existing-chrome-extension-to-blazor-wasm-2
description: "Throughout this post, I'm going to walk through how to migrate the existing Chrome extension with minimal code changes."
date: "2022-07-20"
author: Justin-Yoo
tags:
- dotnet
- blazor-wasm
- chrome-extension
- jsinterop
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2022/07/lift-and-shift-existing-chrome-extension-to-blazor-wasm-00.png
fullscreen: true
---

In my [previous post][post 1], I've walked through how to migrate a JavaScript-based [Chrome extension][chrome extension] to [Blazor WASM][blazor wasm] with minimal code changes. Although it's successfully migrated to Blazor WASM, it doesn't fully use the [JavaScript Interoperability (JS interop)][blazor wasm jsinterop] feature, which is the powerful feature of Blazor WASM. I'm going to take this feature throughout this post.

> You can download the sample app codes from [this GitHub repository][gh sample].


## Chrome Extension &ndash; Before JS Interop ##

The `index.html` file written in the [previous post][post 1] looks like the following. It loads `blazor.webassembly.js` first with the `autostart="false"` option, followed by loading `js/main.js` through the function call. The `js/main.js` reference is replaced with `js/options.js` or `js/popup.js` during the artifact generation process.

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

I'm not happy with this JS loading due to the two reasons below:

1. I have to explicitly give the option of `autostart="false"` while loading the `blazor.webassembly.js` file, which is extra to the bootstrapper.
2. I have to append `js/main.js` through the Promise pattern after `Blazor.start()`, which is another extra point to the bootstrapper.

Can we minimise this modification from the original `index.html` file and use more JS Interop capabilities here so that it can be more Blazor-ish?


## Chrome Extension &ndash; JS Interop Step #1 ##

Let's update the `index.html` file. Unlike the previous update, remove the JS part calling the `Blazor.start()` function. Load `js/main.js` before loading `blazor.webassembly.js`. Remove the `autostart="false"` attribute as well.

```html
<!DOCTYPE html>
<html lang="en">
...
<body>
    <div id="app">Loading...</div>
    ...
    <!-- ⬇️⬇️⬇️ Add this line ⬇️⬇️⬇️ -->
    <script src="js/main.js"></script>
    <!-- ⬆️⬆️⬆️ Add this line ⬆️⬆️⬆️ -->
    <script src="_framework/blazor.webassembly.js"></script>
</body>
</html>
```

Originally the `js/main.js` file was blank, but this time let's add the following JS function that appends another `script` tag to load the given JS file reference.

```javascript
function loadJs(sourceUrl) {
  if (sourceUrl.Length == 0) {
    console.error("Invalid source URL");
    return;
  }

  var tag = document.createElement('script');
  tag.src = sourceUrl;
  tag.type = "text/javascript";

  tag.onload = function () {
    console.log("Script loaded successfully");
  }

  tag.onerror = function () {
    console.error("Failed to load script");
  }

  document.body.appendChild(tag);
}
```

Update the `Popup.razor` file like below:

1. Add the `@inject` declaration for the `IJSRuntime` instance as a dependency.
2. Call the `JS.InvokeVoidAsync` method to load the `js/popup.js` file by invoking the `loadJs` function from the `js/main.js` file.

```razor
@* Popup.razor *@

@page "/popup.html"

@* Inject IJSRuntime instance *@
@inject IJSRuntime JS

...

@code {
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (!firstRender)
        {
            return;
        }
        
        var src = "js/popup.js";

        // Invoke the `loadJs` function
        await JS.InvokeVoidAsync("loadJs", src).ConfigureAwait(false);
    }
}
```

Update the `Options.razor` file in the same way.

```razor
@* Options.razor *@

@page "/options.html"

@* Inject IJSRuntime instance *@
@inject IJSRuntime JS

...

@code {
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (!firstRender)
        {
            return;
        }
        
        var src = "js/options.js";

        // Invoke the `loadJs` function
        await JS.InvokeVoidAsync("loadJs", src).ConfigureAwait(false);
    }
}
```

Now, we don't need the reference replacement part in the PowerShell script. Let's comment them out.

```powershell
# Run-PostBuild.ps1

...

# Update-FileContent `
#     -Filename "./published/wwwroot/popup.html" `
#     -Value1 "js/main.js" `
#     -Value2 "js/popup.js"

# Update-FileContent `
#     -Filename "./published/wwwroot/options.html" `
#     -Value1 "js/main.js" `
#     -Value2 "js/options.js"
```

Build and publish the Blazor WASM app, then run the PowerShell script to get ready for the extension loading. Once reload the extension, it works with no issue. The `loadJs` function is the key that takes advantage of the JS Interop feature.

However, I'm still not happy with adding `js/main.js` to `index.html`. Can we also remove this part from the file and use the JS Interop feature instead?


## Chrome Extension &ndash; JS Interop Step #2 ##

Let's get `index.html` back to the original state when a bootstrapper creates the file. Then, all we can see in `index.html` is the `blazor.webassembly.js` file reference.

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
    <title>ChromeExtensionV2</title>
    <base href="/" />
    <link href="css/bootstrap/bootstrap.min.css" rel="stylesheet" />
    <link href="css/app.css" rel="stylesheet" />
    <link href="ChromeExtensionV2.styles.css" rel="stylesheet" />
</head>

<body>
    <div id="app">Loading...</div>

    <div id="blazor-error-ui">
        An unhandled error has occurred.
        <a href="" class="reload">Reload</a>
        <a class="dismiss">🗙</a>
    </div>
    <script src="_framework/blazor.webassembly.js"></script>
</body>
</html>
```

Then, add the `export` declaration in front of the `loadJs` function in the `js/main.js` file.

```javascript
export function loadJs(sourceUrl) {
...
}
```

Update `Popup.razor` like below.

```csharp
...
var src = "js/popup.js";

// Import the `js/main.js` file
var module = await JS.InvokeAsync<IJSObjectReference>("import", "./js/main.js").ConfigureAwait(false);

// Invoke the `loadJs` function
await module.InvokeVoidAsync("loadJs", src).ConfigureAwait(false);
```

The same change should also be applicable to `Options.razor`. And finally, update the `manifest.json` below because we no longer need the hash key for `popup.js` and `options.js`.

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

Build and publish the app, and run the PowerShell script against the artifact. Then, reload the extension, and you will see the same result.

---

So far, we've walked through how to take more advantage of the [JS Interop][blazor wasm jsinterop] feature that [Blazor WASM][blazor wasm] offers to migrate the existing Chrome extension to Blazor WASM. What could be the potential merits of this exercise?

1. We never touch any bootstrapper codes that Blazor WASM generate for us.
2. If necessary, we load JavaScript for each page using the JS Interop feature. During this practice, C# handles all the JS codes.

If we use more JS Interop features, we can build the Blazor WASM app more effectively, which will be another option for building Chrome extensions.


## Do you want to know more about Blazor? ##

Here are some tutorials for you.

* [Blazor][blazor]
* [Blazor Tutorial][blazor tutorial]
* [Blazor Learn][blazor learn]


[post 1]: /2022/07/08/lift-and-shift-existing-chrome-extension-to-blazor-wasm/
[post 2]: /2022/07/20/lift-and-shift-existing-chrome-extension-to-blazor-wasm-2/

[gh sample]: https://github.com/devkimchi/blazor-wasm-chrome-extension/tree/the-integration
[gh sample v2 blazor]: https://github.com/devkimchi/blazor-wasm-chrome-extension/tree/the-integration/src/ChromeExtensionV2

[chrome extension]: https://developer.chrome.com/docs/extensions/
[chrome extension v2]: https://developer.chrome.com/docs/extensions/mv2/getstarted/

[blazor]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor?WT.mc_id=dotnet-70466-juyoo
[blazor tutorial]: https://dotnet.microsoft.com/learn/aspnet/blazor-tutorial/intro?WT.mc_id=dotnet-70466-juyoo
[blazor learn]: https://docs.microsoft.com/learn/paths/build-web-apps-with-blazor/?WT.mc_id=dotnet-70466-juyoo
[blazor wasm]: https://docs.microsoft.com/aspnet/core/blazor/host-and-deploy/webassembly?view=aspnetcore-6.0&WT.mc_id=dotnet-70466-juyoo
[blazor wasm jsinterop]: https://docs.microsoft.com/aspnet/core/blazor/javascript-interoperability/?view=aspnetcore-6.0&WT.mc_id=dotnet-70466-juyoo
