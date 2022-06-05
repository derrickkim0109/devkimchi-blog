---
title: "Efficient OAuth Authorisation Management in Azure API Management"
slug: apim-token-store
description: "Throughout this post, I'm going to discuss how efficiently manage OAuth authorisations using Azure API Management."
date: "2022-06-10"
author: Justin-Yoo
tags:
- azure
- api-management
- oauth-token-store
- authorisations
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-00.png
fullscreen: true
---

While developing an application, the majority of the time, you use API calls to send and receive messages. Therefore, you should follow an authentication and authorisation process to use the API unless it's the public API. Those authentication/authorisation process uses either 1) an auth key or 2) an access token through an OAuth process.

Using the auth key approach, you can store it in a safe place like the [Azure Key Vault][az kv] service and fetch the key. It's a relatively simple process. However, if you need to use the OAuth process, it gets complicated with the following authorisation process. Here's the conceptual OAuth process.

1. Request an authorisation code
2. Receive the authorisation code
3. Request an access token by providing the authorisation code
4. Receive the access token and a refresh token
5. Call API request by providing the access token
6. Receive the API response
7. Request a new access token by providing the refresh token once the existing access token expires
8. Receive the new access token and a new refresh token
9. Call API request by providing the new access token
10. Receive the API response

![Conceptual OAuth Flow][image-01]

> If you want to know more about the OAuth auth process, please refer to [this doc][oauth flow concept].

Therefore, developers should implement those processes as a part of the application development. You might be lucky if the service you want to use offers an SDK. If not, you should do it all by yourself, which is very cumbersome. What if someone does all the tedious steps for you? Let's say, within the secure place, someone does all the auth process on your behalf and simply returns the access token. If this happens, your application development velocity will increase significantly. [Azure API Management (APIM)][az apim] has recently released a preview feature called "Authorisations" that does the OAuth process on your behalf. Throughout this post, I'm going to discuss this feature using a [Blazor Web Assembly (WASM) app][blazor wasm] hosted on [Azure Static Web Apps (SWA)][az swa].

> You can find the sample apps used in this post at [this GitHub repository][gh sample].

There are two apps in the repository &ndash; one based on [Blazor WASM][blazor wasm] and the other based on React. I'm going to use the Blazor WASM sample here. If you're interested in the React app sample, please visit my colleague, [Aaron Powell][tw aaron]'s [blog post][post 1].


## Blazor Web Assembly App ##

Please refer to the [Blazor Tutorial][blazor tutorial] document for the Blazor WASM in general. Instead, I'm going to take a look at a component taking care of the OAuth authorisation and access token. This component takes the user inputs, stores them as a .csv file format and uploads it to DropBox. The Razor code below is nothing special but contains a form for user inputs. When a user completes the form and clicks the "**Submit**" button, the `OnFormSubmittedAsync` event is triggered (line #3).

```razor
<div class="container-sm" style="max-width: 540px;">
  <h1>Blazor Lead Capture</h1>
  <form class="clearfix" @onsubmit="OnFormSubmittedAsync">
    <fieldset>
      <div>
        <label for="firstName" class="form-label">First name</label>
        <input type="text" class="form-control" id="firstName" name="firstName" placeholder="Justin" value="@userInfo.FirstName" @onchange="@(e => OnFieldChanged(e, "firstName"))" />
      </div>
      <div>
        <label for="lastName" class="form-label">Last name</label>
        <input type="text" class="form-control" id="lastName" name="lastName" placeholder="Yoo" value="@userInfo.LastName" @onchange="@(e => OnFieldChanged(e, "lastName"))" />
      </div>
    </fieldset>

    <fieldset>
      <div>
        <label htmlFor="email" class="form-label">Email</label>
        <input type="email" class="form-control" id="email" name="email" placeholder="bar@email.com" value="@userInfo.Email" @onchange="@(e => OnFieldChanged(e, "email"))" />
      </div>
      <div>
        <label htmlFor="phone" class="form-label">Phone</label>
        <input type="phone" class="form-control" id="phone" name="phone" placeholder="555-555-555" value="@userInfo.Phone" @onchange="@(e => OnFieldChanged(e, "phone"))" />
      </div>
    </fieldset>

    <fieldset>
      <button type="submit" class="btn btn-@componentUIInfo.ButtonColour" disabled="@(componentUIInfo.Submitting || string.IsNullOrWhiteSpace(userInfo.FirstName) || string.IsNullOrWhiteSpace(userInfo.LastName) || string.IsNullOrWhiteSpace(userInfo.Email) || string.IsNullOrWhiteSpace(userInfo.Phone))">
        <span>Submit</span>
        <span class="spinner-border spinner-border-sm" style="display:@componentUIInfo.DisplaySpinner;" role="status" aria-hidden="true"></span>
      </button>
    </fieldset>
  </form>

  <div class="alert alert-@componentUIInfo.AlertResult" style="display:@componentUIInfo.DisplayResult;">
    <h2>@componentUIInfo.MessageResult</h2>
    <button type="reset" class="btn btn-dark" @onclick="ResetFields">
      <span>Start Over?</span>
    </button>
  </div>
</div>
```

The following `@code { ... }` block is the C# code interacting with the Razor component. For brevity, I've left two methods. The first method is the event handler, `OnFormSubmittedAsync` triggered by the "**Submit**" button. Then, inside the event handler, it calls the `SaveToDropboxAsync` method, which takes care of all the things, including access token fetch and DropBox save.

```csharp
@code {
    ...
    protected async Task OnFormSubmittedAsync(EventArgs e)
    {
        ...
        await SaveToDropboxAsync().ConfigureAwait(false);
    }
```

In the `SaveToDropboxAsync` method, it firstly gets the environment variable of `APIM_Endpoint` (line #4). Blazor WASM stores all the environment variables in the `appsettings.json` file, which I will discuss later. By calling the APIM endpoint from `appsettings.json`, the app gets the access token to DropBox (line #7), and the DropBox client instance uploads the data.

```csharp
    private async Task SaveToDropboxAsync()
    {
        // Gets the APIM endpoint from appsettings.json
        var requestUrl = Configuration.GetValue<string>("APIM_Endpoint");
        
        // Gets the auth token from APIM
        var token = await Http.GetStringAsync(requestUrl).ConfigureAwait(false);

        // Builds contents.
        var path = $"/submissions/{DateTimeOffset.UtcNow.ToString("yyyyMMddHHmmss")}.csv";
        var contents = $"{userInfo.FirstName},{userInfo.LastName},{userInfo.Email},{userInfo.Phone}";
        var bytes = UTF8Encoding.UTF8.GetBytes(contents);

        // Uploads the contents.
        var result = default(FileMetadata);
        using(var dropbox = new DropboxClient(token))
        using(var stream = new MemoryStream(bytes))
        {
            result = await dropbox.Files.UploadAsync(path, WriteMode.Overwrite.Instance, body: stream).ConfigureAwait(false);
        }

        ...
    }
}
```

Let's take a look at the `appsettings.json` file that contains the `APIM_Endpoint` environment variable. The value is the APIM endpoint to get the DropBox access token.

```json
{
  "APIM_Endpoint": "https://<APIM_NAME>.azure-api.net/dropbox-demo/token?subscription-key=<APIM_SUBSCRIPTION_KEY>"
}
```

Run the Blazor WASM app on your local machine.

```powershell
dotnet watch run
```

Then, open your web browser and enter the URL of `https://localhost:5001`, and you will see the page like:

![Blazor WASM Landing Page][image-02]

Fill out the form and click the "**Submit**" button, and the app will save the form details to DropBox. Then, if you open the Developer Tools of your web browser, you can see how the Blazor WASM app calls the APIM endpoint.

![Request Access Token #1][image-03]

And the API call returns the access token to DropBox.

![Request Access Token #2][image-04]

With this access token, you can create and store a file. Here's the result:

![Dropbox Upload Result][image-05]

There are no codes for OAuth authorisation to DropBox, as you can recall. Instead, it starts from the API call that directly gets the access token. So how can the code remove all the preflight codes before getting the access token? This is the new APIM authorisation feature being referred to in this post. In short, the APIM instance internally performs all the OAuth related processes on behalf of the Blazor WASM app and simply returns the access token.


## Azure API Management Instance ##

Let's dig into the new APIM feature. You can click the button below to provision all the resources onto Azure at once.

[![Deploy To Azure](https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/1-CONTRIBUTION-GUIDE/images/deploytoazure.svg?sanitize=true)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Faaronpowell%2Ftoken-store-demo%2Fmain%2Fsrc%2Fbackend%2Fmain.json)

Alternatively, you can run the bicep files declared below. Let's take a further look at the bicep files. First, declare the APIM instance. You might notice that it enables the [Managed Identity][az apim mi] feature (line #13-15), which I'll discuss later in this post. For convenience, name the APIM instance of `token-store-demo-apim` and set the location to `West Central US`. The resource group for this provision is set to `rg-token-store-demo`.

```javascript
// APIM instance
resource apim 'Microsoft.ApiManagement/service@2021-08-01' = {
  name: 'token-store-demo-apim'
  location: 'westcentralus'
  sku: {
    name: 'Developer'
    capacity: 1
  }
  properties: {
    publisherName: 'John Doe'
    publisherEmail: 'john.doe@nomail.com'
  }
  identity: {
    type: 'SystemAssigned'
  }
}
```

The next step is the CORS policy because Azure SWA directly calls the APIM endpoint. Within the `inbound` node, add the `cors` node, add the `allowed-origins` node under it, and add the `origin` node with its value of `*`. Make sure that it's just for the demo purpose. You should add the specific URL for better security if it goes live.

```javascript
// Service Policy
resource apim_policy 'Microsoft.ApiManagement/service/policies@2021-08-01' = {
  parent: apim
  name: 'policy'
  properties: {
    value: service_policy
    format: 'xml'
  }
}

// Service Policy Definition
var service_policy = '''
<policies>
    <inbound>
        <cors allow-credentials="false">
            <allowed-origins>
                <origin>*</origin>
            </allowed-origins>
            <allowed-methods>
                <method>GET</method>
                <method>POST</method>
            </allowed-methods>
        </cors>
    </inbound>
    <backend>
        <forward-request />
    </backend>
    <outbound />
    <on-error />
</policies>'''
```

After declaring the APIM instance, define an API and its operation. The API's `serviceUrl` is set to the base URL of the DropBox API, and the operation endpoint is set to `/token` that returns the access token. Therefore, the entire APIM endpoint might look like `https://token-store-demo-apim.azure-api.net/dropbox-demo/token`.

```javascript
// API
resource api 'Microsoft.ApiManagement/service/apis@2021-08-01' = {
  name: 'dropbox-demo'
  parent: apim
  properties: {
    serviceUrl:'https://api.dropboxapi.com'
    path: 'dropbox-demo'
    displayName:'dropbox-demo'
    protocols:[
      'https'
    ]
  }
}

// Operation
resource api_gettoken 'Microsoft.ApiManagement/service/apis/operations@2021-08-01' = {
  name: 'gettoken'
  parent: api
  properties: {
    method: 'GET'
    urlTemplate: '/token'
    displayName: 'gettoken'
  }
}
```

However, this endpoint doesn't exist on DropBox API. Therefore, add the operation policy to make this operation work.

```javascript
// Operation Policy
resource api_gettoken_policy 'Microsoft.ApiManagement/service/apis/operations/policies@2021-08-01' = {
  parent: api_gettoken
  name: 'policy'
  properties: {
    value: operation_token_policy
    format: 'xml'
  }
}
```

The following XML document of the operation policy explains the core idea of this APIM's new feature. Add the `get-authorization-context` node under `inbound`. It has the following attributes.

* `provider-id`: `dropbox-demo`
* `authorization-id`: `auth`
* `context-variable-name`: `auth-context`
* `identity-type`: `managed`

As you can see, the APIM instance has enabled the Managed Identity feature corresponding to `identity-type`. Both `provider-id` and `authorisation-id` will be used later. The `context-variable-name` attribute is set to `auth-context`. It's used in the `return-response` node that holds the access token value. Overall, this operation policy takes care of getting access token on behalf of the SWA app.

```javascript
// Operation Token Policy Definition
var operation_token_policy = '''
<policies>
    <inbound>
        <base />
        <get-authorization-context provider-id="dropbox-demo" authorization-id="auth" context-variable-name="auth-context" ignore-error="false" identity-type="managed" />
        <return-response>
            <set-body>@(((Authorization)context.Variables.GetValueOrDefault(&quot;auth-context&quot;))?.AccessToken)</set-body>
        </return-response>
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>'''
```

You've got the APIM instance defined. Now, you need the SWA app instance for the Blazor WASM app.


## Azure Static Web Apps Instance ##

To host the Blazor WASM app, you need to provision an [Azure SWA][az swa] instance. Here's the bicep code for it. First, give the app name `token-store-demo-blazor-swa` and the location of `Central US` under the resource group of `rg-token-store-demo`.

```javascript
// SWA instance
resource sttapp 'Microsoft.Web/staticSites@2021-02-01' = {
  name: 'token-store-demo-blazor-swa'
  location: 'centralus'
  sku: {
    name: 'Free'
  }
  properties: {
    allowConfigFileUpdates: true
    stagingEnvironmentPolicy: 'Enabled'
  }
}
```


## Blazor WASM App Deployment to Azure Static Web Apps Instance ##

You've got the new ASWA instance. It's time to deploy the Blazor WASM app, which is located in the `src/frontend/blazor` directory. The first step should be adding the `appsettings.json` file that contains the APIM endpoint for the access token. The following commands are how to get the APIM's endpoint URL.

```powershell
# Get APIM gateway URL
rg_name=rg-token-store-demo
apim_name=token-store-demo-apim

gateway_url=$(az apim show -g $rg_name -n $apim_name --query "gatewayUrl" -o tsv)

# Get APIM subscription key
subscription_id=$(az account show --query "id" -o tsv)
apim_secret_uri=/subscriptions/$subscription_id/resourceGroups/$rg_name/providers/Microsoft.ApiManagement/service/$apim_name/subscriptions/master/listSecrets
api_version=2021-08-01

subscription_key=$(az rest --method post --uri $apim_secret_uri\?api-version=$api_version | jq '.primaryKey' -r)

# Build APIM endpoint
apim_endpoint=$gateway_url/dropbox-demo/token\?subscription-key=$subscription_key
```

The APIM's endpoint URL is finally landed in the `apim_endpoint` variable. Therefore, rename the `appsettings.sample.json` file under the `src/frontend/blazor/wwwroot` directory to `appsettings.json` and update the endpoint.

```json
{
  "APIM_Endpoint": "<apim_endpoint>"
}
```

Build and create the artifact of the Blazor WASM app.

```powershell
dotnet restore ./src/frontend/blazor
dotnet build ./src/frontend/blazor
dotnet publish ./src/frontend/blazor -c Release -o ./src/frontend/blazor/bin
```

There's one more step for deployment. Get the deployment key by running the command below:

```powershell
swa_key=$(az staticwebapp secrets list \
    -g rg-token-store-demo \
    -n token-store-demo-blazor-swa \
    --query "properties.apiKey" -o tsv)
```

Finally, deploy the Blazor WASM app by running the command.

```powershell
swa deploy -a ./src/frontend/blazor/bin/wwwroot -d $swa_key --env default
```

Although everything has gone perfectly so far, you can see the error below after submitting the form:

![Internal Server Error][image-06]

It's because you haven't made consent to the DropBox app yet. The final step will be consent.


## DropBox App Consent ##

Create a DropBox app by following [this document][dropbox appconsole], and you will get both `App key` and `App secret`.

![Dropbox App][image-07]

You need both values for the consent within the APIM instance. Click the "**Authorisations (preview)**" at the blade.

![Authorizations Preview][image-08]

There are no authorisation apps yet. Therefore, click the "**Create**" button.

![Authorizations Preview Pane][image-09]

You can recall how you set up the `get-authorisation-context` node for the operation policy. It's time to use them.

* Enter `dropbox-demo` to the "**Provider name**" field.
* Select `DropBox` from the "**Identity provider**" field.
* Enter DropBox's app key value to the "**Client id**" field.
* Enter DropBox's app secreet value to the "**Client secret**" field.
* Enter `files.metadata.write files.content.write files.content.read` to the "**Scopes**" field.
* Enther `auth` to the "**Authorization name**" field.

![Create Authorization][image-10]

Click the "**Create**" button and get the redirection URL.

![Redirection URL][image-11]

Add the redirection URL to the DropBox app.

![DropBox App Update][image-12]

Back to the APIM instance and log in to the DropBox app. Then, proceed the pop-up window by clicking the "**Continue**" and "**Allow**" and "**Allow access**" buttons consecutively.

![DropBox App Login on APIM][image-13]

Then, you will see the success message.

![DropBox App Authorized][image-14]

The DropBox app has now been authorised. But the APIM instance has not yet been authorised to access the DropBox app. Since the APIM instance has the Managed Identity feature enabled, let's use it. Choose "**Managed identity**" and click the "**Add members**" button.

![APIM Managed Identity][image-15]

Find the APIM instance and add it.

![Add APIM Managed Identity][image-16]

Now, both APIM and DropBox can communicate with each other.

![APIM Managed Identity Added][image-17]

Can you see the DropBox app authorised?

![DropBox Authorization Created][image-18]

Let's test the endpoint whether it actually works or not. Follow the menu "**APIs**" ➡️ "**dropbox-demo**" ➡️ "**gettoken**" and click the "**Send**" button.

![APIM Test][image-19]

Then you will see the access token successfully issued.

![APIM Test Success][image-20]

Let's get back to the ASWA app and fill out the form. There is no error this time.

![Static Web App to APIM Request Success][image-21]

And the uploaded file appears on DropBox!

![DropBox File Created][image-22]

---

So far, we've discussed the new OAuth authorisation feature of [Azure API Management][az apim]. Ultimately, we need the access token through the OAuth process. APIM performs this process. Therefore, we can save a huge amount of time by not implementing this feature within our app. Since it's currently in preview, some features may be ready, but they will be continuously improved.

If you want to know more about this APIM OAuth authorisation management feature, please visit the document below:

* [Azure API Management Authorisations Management][az apim tokenstore]



[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-02.png
[image-03]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-03.png
[image-04]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-04.png
[image-05]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-05.png
[image-06]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-06.png
[image-07]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-07.png
[image-08]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-08.png
[image-09]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-09.png
[image-10]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-10.png
[image-11]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-11.png
[image-12]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-12.png
[image-13]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-13.png
[image-14]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-14.png
[image-15]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-15.png
[image-16]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-16.png
[image-17]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-17.png
[image-18]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-18.png
[image-19]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-19.png
[image-20]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-20.png
[image-21]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-21.png
[image-22]: https://sa0blogs.blob.core.windows.net/devkimchi/2022/06/apim-token-store-22.png


[post 1]: .

[gh sample]: https://github.com/aaronpowell/token-store-demo

[tw aaron]: https://twitter.com/slace

[blazor]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor?WT.mc_id=dotnet-57408-juyoo
[blazor wasm]: https://docs.microsoft.com/aspnet/core/blazor/host-and-deploy/webassembly?view=aspnetcore-6.0&WT.mc_id=dotnet-57408-juyoo
[blazor tutorial]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor?WT.mc_id=dotnet-57408-juyoo

[az swa]: https://docs.microsoft.com/azure/static-web-apps/overview?WT.mc_id=dotnet-57408-juyoo

[az apim]: https://docs.microsoft.com/azure/api-management/api-management-key-concepts?WT.mc_id=dotnet-57408-juyoo
[az apim mi]: https://docs.microsoft.com/azure/api-management/api-management-howto-use-managed-service-identity?WT.mc_id=dotnet-57408-juyoo
[az apim tokenstore]: .

[az kv]: https://docs.microsoft.com/azure/key-vault/general/basic-concepts?WT.mc_id=dotnet-57408-juyoo

[oauth flow concept]: https://docs.microsoft.com/azure/active-directory/develop/v2-oauth2-auth-code-flow?WT.mc_id=dotnet-57408-juyoo

[dropbox appconsole]: https://www.dropbox.com/developers/reference/getting-started#app%20console
