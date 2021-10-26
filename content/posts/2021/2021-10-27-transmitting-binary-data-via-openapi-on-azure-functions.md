---
title: "Transmitting Binary Data via OpenAPI on Azure Functions"
slug: transmitting-binary-data-via-openapi-on-azure-functions
description: "In this post, I'm going to discuss how to declare byte array data type to transfer binary data through Azure Functions OpenAPI extension."
date: "2021-10-27"
author: Justin-Yoo
tags:
- azure-functions
- openapi
- binary-data
- byte-array
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/10/transmitting-binary-data-via-openapi-on-azure-functions-00.png
fullscreen: true
---

Since the [latest version (0.9.0-preview)][az fncapp openapi nuget] of the [OpenAPI extension][az fncapp openapi] for [Azure Functions][az fncapp] was released, it supports the byte array types. With this support, you can now define the binary data type like image files onto the OpenAPI document. Throughout this post, I'm going to discuss how to declare the binary data and transfer it through the Azure Functions OpenAPI extension.

> You can download the sample app code from [this GitHub repository][gh sample].


## Binary Data Transfer with `text/plain` Content Type ##

Let's look at the function below. It directly passes the based64 encoded string to Azure Function app through the request payload. It would be best to assume that the binary data has already been converted to a base64 string. The function defines the content type of `text/plain` and data type of `byte[]` *(line #7)*. For the response payload, it defines the content type of `image/png` and data type of `byte[]` *(line #9)*. As the actual data you are passing is the base64 encoded string, all you need to do is to convert the encoded string into byte array *(line #15-21)*.

https://gist.github.com/justinyoo/9d875913235af8b7636810e888fb0013?file=01-run-byte-array.cs&highlights=7,9,15-21

The OpenAPI document about this part looks like below (omitted unrelated lines for brevity). First, the request payload is defined as `text/plain`, with the data type of `string` and format of `binary` *(line #7-10)*. Next, the response payload is also defined as `image/png`, with the data type of string and format of `binary` *(line #16-19)*.

https://gist.github.com/justinyoo/9d875913235af8b7636810e888fb0013?file=02-run-byte-array.yaml&highlights=7-10,16-19

Once you run the function app on your local machine, it looks like below. The image data is transferred correctly.

![Byte Array][image-01]


## Binary Data Transfer with `multipart/form-data` Content Type ##

Let's transfer both binary data and text data through the `multipart/form-data` content type. This content type is the most typical way to transfer binary data through API. The request payload declares the content type of `multipart/form-data` and data type of `MultiPartFormDataModel` *(line #7)*. `MultiPartFormDataModel` is a DTO that contains the `Image` property with the byte array type *(line #34)*. When you send a POST request to the API endpoint, because you use the `multipart/form-data` content type, the image data should be retrieved from `req.Form.Files[0]`, and you need to convert it to the byte array *(line #15-23)*.

https://gist.github.com/justinyoo/9d875913235af8b7636810e888fb0013?file=03-run-multi-part-form-data.cs&highlights=7,15-23,35

The OpenAPI document related to this part looks like below (omitted unrelated lines for brevity). The request payload has a reference to the object *(line #9)*, and the object has a property of `image`, with the data type of `string` and format of `binary` *(line #31-33)*.

https://gist.github.com/justinyoo/9d875913235af8b7636810e888fb0013?file=04-run-multi-part-form-data.yaml&highlights=9,31-33

Run the function app and see how it's going. Your image data has been transferred successfully through the `multipart/form-data` content type.

![Multi-Part Form-Data][image-02]

---

So far, we've walked through how to define binary data through Azure Functions OpenAPI extension and run it on Swagger UI. Since this feature was one of the long-waited ones, I'm hoping everyone can make use of this feature in many places.


[image-01]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/10/transmitting-binary-data-via-openapi-on-azure-functions-01.png
[image-02]: https://sa0blogs.blob.core.windows.net/devkimchi/2021/10/transmitting-binary-data-via-openapi-on-azure-functions-02.png


[gh sample]: https://github.com/devkimchi/azure-functions-binary-data-via-swagger-ui

[az fncapp]: https://docs.microsoft.com/azure/azure-functions/functions-overview?WT.mc_id=dotnet-47576-juyoo
[az fncapp openapi]: https://aka.ms/azfunc-openapi
[az fncapp openapi nuget]: https://www.nuget.org/packages/Microsoft.Azure.WebJobs.Extensions.OpenApi
