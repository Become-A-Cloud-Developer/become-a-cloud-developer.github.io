+++
title = "Display Images from Azure Blob Storage in an ASP.NET Web Application"
weight = 6
date = 2024-03-04
draft = false
+++

## Introduction

In this tutorial, we're going to guide you through the steps to display images stored in Azure Blob Storage on an ASP.NET web application. This process involves setting up Blob Storage with public access, uploading images to it, and then configuring an ASP.NET Razor Pages web application to retrieve and display these images.

## Method

- We will utilize the Azure portal to configure a Blob Storage container, which is anonymously accessible. 
- An ASP.NET Razor Pages web application will be developed to serve our images.
- Within the web application, we will develop an image service in order to bridge the gap between our Azure Blob Storage and the ASP.NET web application. The service will be responsible for fetching the URLs of the images stored in the Blob Storage, using the Azure Storage Blobs client library for .NET.
- The configurations, such as the connection string and container name, will be loaded into our web application’s settings via environment variables.

## Prerequisites

- An Azure account. If you don't have one, sign up at [Azure's official site](https://azure.microsoft.com/).
- Basic familiarity with Azure services and ASP.NET web development.

## Step 1: Set Up Azure Blob Storage

1. Create a Resource Group `BlobDemoRG`
2. Create a Storage Account:
	- Select **Storage accounts** in the resource menu
	- Click the **+ Create** button
		- Resource Group: `BlobDemoRG`
		- Storage account name: `blobdemo<datetime>` (change datetime to the current date and time)
		- Press **Review**
		- Press **Create**
		- Press **Go to resource**
	- Select **Configuration** in the _Settings_ section of the storage account menu
		- (Alt: Select **Security** from storage account _Overview_)
	- Enable **Allow Blob anonymous access**
	- Press **Save**
3. Create a Blob Storage:
	- Select **Container** in the _Data storage_ section of the storage account menu
	- Press **+ Container**
		- Choose a name: `imagerepository`
		- Anonymous access level: `Blob (anonymous read access for blobs only)`
		- Press **Create**
4. Upload images
	- Select the `imagerepository`
	- Press **Upload**
	- Select the images and press **Upload**

### Verify the image upload
	
- Select an image to open up details
- Copy the URL and open in a private browser window

> It´s important that it is a private browser window since you are logged in to the Azure portal in the regular browser window. This will grant you access also to images that are not necessarily publically accessible on the Internet


### Retrieve the Connection String

- Select your newly created blob storage `blobdemo<datetime>`
- Go to **Access keys** in the _Security + networking_ section of the storage account menu
- Find _Connection string_ and press **Show**. Copy the _Connection string_ and use it in your app

## Step 2: Create the Web Application

### Set Up the Webapp

Create a directory called `BlobStorage` and open up a terminal. Run the command below to create a Razor Pages webapp:

```bash
dotnet new webapp
```

Verify:

```bash
dotnet run
```

### Develop An Image Service

#### Prepare the appsettings

> appsettings.json

```json
...

  "AllowedHosts": "*",
  "AzureBlobImageService": {
    "ConnectionString": "Set ConnectionString in environment variable or user secrets",
    "ContainerName": "Set ContainerName in environment variable or user secrets"
  }
}
```

#### Prepare a `.env` file

> .env

```bash
# Make sure this file is not checked into source control!

# Use the following commands to export the environment variables to the current shell
# set -a
# source ./.env
# set +a

# You can also use dotnet user-secrets to set the environment variables
# dotnet user-secrets set "BlobStorageSettings__ConnectionString" "ConnectionString"
# dotnet user-secrets set "BlobStorageSettings__ContainerName" "Container"
# dotnet user-secrets list
# dotnet user-secrets clear

# Note the double underscore (__) to indicate hierarchi in environment variables

AzureBlobImageService__ConnectionString="Paste in the connection string here"
AzureBlobImageService__ContainerName="Paste in the container name here"

```

Run the .env file in order to set environment variables

```bash
set -a
source ./.env
set +a
```
Verify the environment variables by running:

```bash
export
```

#### Add Nuget package for Azure Blob Storage

```bash
dotnet add package Azure.Storage.Blobs
```


#### Develop a Service Interface

Create a new directory `Services`.

Create a new file in the `Services` directory called `IImageService.cs`

> /Services/IImageService.cs

```csharp
using System.Threading.Tasks;

namespace BlobStorage.Services;

public interface IImageService
{
    public Task<List<string>> GetAllImageUrlsAsync();
}
```


#### Develop a service implementation using Azure Blob Storage

Create a new file in the `Services` directory called `AzureBlobImageService.cs`

> /Services/AzureBlobImageService.cs

```csharp
using Azure.Storage.Blobs;

namespace BlobStorage.Services;

public class AzureBlobImageService : IImageService
{
    private readonly BlobContainerClient _containerClient; // Used to interact with the Azure Blob Storage Container

    public AzureBlobImageService(IConfiguration configuration)
    {
        // Get the configuration settings from dependency injection
        var blobServiceConfig = configuration.GetSection("AzureBlobImageService");
        string connectionString = blobServiceConfig.GetValue<string>("ConnectionString") ?? throw new InvalidOperationException("ConnectionString configuration is missing.");
        string containerName = blobServiceConfig.GetValue<string>("ContainerName") ?? throw new InvalidOperationException("ContainerName configuration is missing.");

        // Create the BlobContainerClient
        var blobServiceClient = new BlobServiceClient(connectionString);            // Connect to the Azure Blob Storage account
        _containerClient = blobServiceClient.GetBlobContainerClient(containerName); // Get a reference to the Container
    }

    public async Task<List<string>> GetAllImageUrlsAsync()
    {
        // Prepare a list to hold the image URLs
        var imageUrls = new List<string>();

        // Loop through the blobs in the container asynchronously (which is the recommended way to interact with Azure Blob Storage)
        await foreach (var blobItem in _containerClient.GetBlobsAsync())
        {
            // Create a BlobClient for each blob
            var blobClient = _containerClient.GetBlobClient(blobItem.Name);

            // Use the blob client to get the URL of the blob and add it to the list
            imageUrls.Add(blobClient.Uri.ToString());
        }

        // Return the list of image URLs
        return imageUrls;
    }
}
```

#### Register the service in Dependeny Injection

> Program.cs

```csharp
using BlobStorage.Services;

...

// Add services to the container.
builder.Services.AddRazorPages();

...

// Register the IImageService with the DI container
builder.Services.AddSingleton<IImageService, AzureBlobImageService>();

...

app.Run();
```

## Present the images on the web page

### Create an OnGetAsync method in the controller

> /Pages/Index.cshtml.cs

```csharp
using BlobStorage.Services;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace BlobStorage.Pages;

public class IndexModel : PageModel
{
    private readonly ILogger<IndexModel> _logger;
    private readonly IImageService _imageService;

    public IndexModel(ILogger<IndexModel> logger, IImageService imageService)
    {
        _logger = logger;
        _imageService = imageService;
    }

    public async Task<IActionResult> OnGetAsync()
    {
        var imageUrls = await _imageService.GetAllImageUrlsAsync();
        ViewData["ImageUrls"] = imageUrls;
        return Page();
    }
}
```

### Update the page to list the images

> /Pages/Index.cshtml

```csharp
@page
@model IndexModel
@{
    ViewData["Title"] = "Home page";
}

<h1>List of Images:</h1>

<ul>
    @* Check if the ViewData has a value and if that value is of type List<string> *@
    @if (ViewData["ImageUrls"] is List<string> imageUrls)
    {
        @* Loop through the list and display the images *@
        @foreach (var imageUrl in imageUrls)
        {
            <li><img src="@imageUrl" alt="Image" width="100"></li>
        }
    }
</ul>
```

### Run the webapp

Verify that you can see a list of images:

```bash
dotnet run
```

## Conclusion

You've successfully learned how to display images from Azure Blob Storage in an ASP.NET web application. This guide introduced you to creating and configuring Blob Storage for public access, uploading images, and setting up an ASP.NET web application to display these images.

## Don't Forget

Azure services incur costs. Delete resources you no longer need.

# Happy Developing! 🚀


## References

This tutorial is based on the following articles

Working with Azure Blob Storage:

https://learn.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-dotnet?tabs=net-cli%2Cmanaged-identity%2Croles-azure-portal%2Csign-in-azure-cli%2Cidentity-visual-studio&pivots=blob-storage-quickstart-scratch

Lifetime:

https://devblogs.microsoft.com/azure-sdk/lifetime-management-and-thread-safety-guarantees-of-azure-sdk-net-clients/































