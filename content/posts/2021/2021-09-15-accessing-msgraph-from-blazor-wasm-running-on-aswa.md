---
title: "Accessing User Profile from Blazor WebAssembly on Azure Static Web Apps"
slug: accessing-msgraph-from-blazor-wasm-running-on-aswa
description: "In this post, I'm going to discuss how to access user profile through MS Graph, from Blazor WebAssembly running on Azure Static Web Apps."
date: "2021-09-15"
author: Justin-Yoo
tags:
- azure-static-web-apps
- blazor-wasm
- msal
- msgraph
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/09/accessing-msgraph-from-blazor-wasm-running-on-aswa-00.png
fullscreen: true
---

[Azure Static Web Apps (ASWA)][az swa] offers a straightforward [authentication][az swa authn] feature. With this feature, you don't need to write a complicating authentication logic by your hand and can sign in to ASWA. By the way, the authentication details from there only show whether you've logged in or not. If you need more information, you should do something more on your end. Throughout this post, I'm going to discuss how to access your user profile data stored in [Azure Active Directory (AAD)][az ad] through [Microsoft Graph][ms graph] from the [Blazor WebAssembly (WASM)][blazor wasm] app running on an [ASWA][az swa] instance.

> You can find the sample code used in this post on this [GitHub repository][gh sample] (docs in Korean).


## Retrieving Authentication Data from Azure Static Web Apps ##

After publishing your [Blazor WASM][blazor wasm] app to [ASWA][az swa], the page before log-in might look like this:

![Before log-in][image-01]

If you want to use [AAD][az ad] as your primary identity provider, add the link to the `Login` HTML element.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=01-auth-login-aad.txt

After the sign-in, you can retrieve your authentication details by calling the API endpoint like below. For brevity, I omitted unnecessary codes.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=02-auth-me.cs

Here are the authentication details from the response:

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=03-auth-details.json

As mentioned above, there's only limited information available from the response. Therefore, if you need more user details, you should do some additional work on your end.


## Accessing User Data through Microsoft Graph ##

You only know your email address used for log-in. Here are the facts about your logged-in details:

* You signed in through your tenant where your email belongs.
* The sign-in information **TELLS** your email address used for log-in.
* The sign-in information **DOESN'T TELL** the tenant information where you logged in.
* The sign-in information **DOESN'T TELL** the tenant information where the ASWA is hosted.
* The sign-in information **DOESN'T TELL** the tenant information where you want to access.

In other words, there are chances that all three tenants details &ndash; the tenant where you logged in, the tenant hosting the ASWA instance, and the tenant where you want to access &ndash; might be different from each other. All you know of my details are:

1. You logged into a tenant, and
2. You only know my email address used for log-in.

Then, how can you know your user details from the tenant that you want to access?

First of all, you need to get permission to get the details to the tenant. Although you signed in to ASWA, it doesn't mean you have enough permission to access the resources. Because [ASWA][az swa] offers [Azure Functions][az swa api] as its facade API, let's use this feature.

When calling the facade API from the Blazor WASM app side, it always includes the auth details through the request header of `x-ms-client-principal`. The information is the Base64 encoded string, which looks like this:

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=04-client-principal.txt

Therefore, decode the string and deserialise it to get the email address for log-in. Here's a POCO class for deserialisation.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=05-client-principal.cs

With this POCO class, deserialise the header value and get the email address you're going to utilise.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=06-get-client-principal.cs

All the plumbing to get the user details is done. Let's move on.


## Registering App on Azure Active Directory ##

The next step is to [register an app][ms al apprego] on [AAD][az ad] through Azure Portal. I'm not going to go further for this step but will give you [this document][ms al apprego] to get it done. Once you complete app registration, you should give it [appropriate roles and permissions][ms al apprego roles], which is the [application permission][ms al apprego roles] instead of the delegate permission.  For example, `User.Read.All` permission should be enough for this exercise.

Once you complete this step, you'll have `TenantID`, `ClientID` and `ClientSecret` information.


## Microsoft Authentication Library (MSAL) for .NET ##

You first need to get an access token to retrieve your details stored on [AAD][az ad]. There are many ways to get the token, but let's use the [client credential][ms al clientcredential] approach for this time. First, as we're using Blazor WASM, we need a NuGet package to install.

* [Microsoft.Identity.Client][nuget msal]: `dotnet add package Microsoft.Identity.Client`

After installing the package, add several environment variables to `local.settings.json`. Here are the details for authentication.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=07-local-settings.json

To get the access token, write the code below. Without having to worry about the user interaction, simply use both ClientID and ClientSecret values, and you'll get the access token. For example, if you use the [`ConfidentialClientApplicationBuilder`][ms al clientcredential builder] class, you'll easily get the one (line #16-20).

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=08-get-access-token.cs&highlights=16-20

Once you have the access token in hand, you can use [Microsoft Graph][ms graph] API.


## Microsoft Graph API for .NET ##

To use Microsoft Graph API, install another NuGet package:

* [Microsoft.Graph][nuget msgraph]: `dotnet add package Microsoft.Graph`

And here's the code to get the Graph API. Call the method written above, `GetAccessTokenAsync()` (line #4-8).

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=09-get-graph-client.cs&highlights=4-8

Finally, call the `GetGraphClientAsync()` method to create the Graph API client (line #1) and get the user details using the email address taken from the `ClientPrincipal` instance (line #4). If no user data is queried, you can safely assume that the email address used for the ASWA log-in is not registered as either a Guest User or an External User. Therefore, the code will return the `404 Not Found` response (line #7).

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=10-get-user.cs&highlights=1,4,7

The amount of your information would be huge if you could filter out your details from AAD.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=11-user.json

You don't want to expose all the details to the public. Therefore, you can create another POCO class only for the necessary information.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=12-loggedin-user.cs

And return the POCO instance to the Blazor WASM app side.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=13-get-loggedin-user.cs

Now, you've got the API to get the user details. Let's keep moving.


## Exposing User Details on Azure Static Web Apps ##

Here's the code that the Blazor WASM app calls the API to get the user details. I use the `try { ... } catch { ... }` block here because I want to silently proceed with the response regardless it indicates success or failure. Of course, You should handle it more carefully, but I leave it for now.

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=14-get-loggedin-user.cs

In your Blazor component, the method `GetLoggedInUserDetailsAsync()` is called like below (line #6, 18).

https://gist.github.com/justinyoo/38ae8fdd9ac6161551a9aee0d15b76e7?file=15-get-loggedin-user-en.razor&highlights=6,18

If your email address belongs to the tenant you want to query, you'll see the result screen like this:

![After the log-in - user found][image-02]

If your email address doesn't belong to the tenant you want to query, you'll see the result screen like this:

![After the log-in - user not found][image-03]

Now, we can access your user details from the Blazor WASM app running on ASWA through Microsoft Graph API.

---

So far, I've walked through the entire process to get the user details:

* Using [Blazor WASM][blazor wasm] app hosted on [ASWA][az swa],
* Using [MSAL][ms al] for authentication and authorisation against [AAD][az ad], and
* Using [Microsoft Graph][ms graph] to access to the user details.

As you know, Microsoft Graph can access all [Microsoft 365][m365] resources like [SharePoint Online][m365 spo], [Teams][m365 teams] and so forth. So if you follow this approach, your chances to use Microsoft 365 resources will get more broadened.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/09/accessing-msgraph-from-blazor-wasm-running-on-aswa-01-en.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/09/accessing-msgraph-from-blazor-wasm-running-on-aswa-02-en.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/09/accessing-msgraph-from-blazor-wasm-running-on-aswa-03-en.png


[gh sample]: https://github.com/fusiondevkr/fusiondevkr

[az swa]: https://docs.microsoft.com/azure/static-web-apps/overview?WT.mc_id=dotnet-42714-juyoo
[az swa authn]: https://docs.microsoft.com/azure/static-web-apps/authentication-authorization?WT.mc_id=dotnet-42714-juyoo
[az swa api]: https://docs.microsoft.com/azure/static-web-apps/apis?WT.mc_id=dotnet-42714-juyoo

[az fncapp]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-42714-juyoo

[blazor wasm]: https://docs.microsoft.com/aspnet/core/blazor/?WT.mc_id=dotnet-42714-juyoo

[az ad]: https://docs.microsoft.com/azure/active-directory/fundamentals/active-directory-whatis?WT.mc_id=dotnet-42714-juyoo

[ms al]: https://docs.microsoft.com/azure/active-directory/develop/msal-overview?WT.mc_id=dotnet-42714-juyoo
[ms al clientcredential]: https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-client-creds-grant-flow?WT.mc_id=dotnet-42714-juyoo#get-a-token
[ms al clientcredential builder]: https://docs.microsoft.com/dotnet/api/microsoft.identity.client.confidentialclientapplicationbuilder?WT.mc_id=dotnet-42714-juyoo
[ms al apprego]: https://docs.microsoft.com/azure/active-directory/develop/quickstart-register-app?WT.mc_id=dotnet-42714-juyoo
[ms al apprego roles]: https://docs.microsoft.com/azure/active-directory/develop/quickstart-configure-app-access-web-apis?WT.mc_id=dotnet-42714-juyoo#application-permission-to-microsoft-graph

[ms graph]: https://docs.microsoft.com/graph/overview?WT.mc_id=dotnet-42714-juyoo

[nuget msal]: https://www.nuget.org/packages/Microsoft.Identity.Client/
[nuget msgraph]: https://www.nuget.org/packages/Microsoft.Graph/

[m365]: https://www.microsoft.com/microsoft-365?WT.mc_id=dotnet-42714-juyoo
[m365 spo]: https://www.microsoft.com/microsoft-365/sharepoint/collaboration?WT.mc_id=dotnet-42714-juyoo
[m365 teams]: https://www.microsoft.com/microsoft-teams/group-chat-software?WT.mc_id=dotnet-42714-juyoo
