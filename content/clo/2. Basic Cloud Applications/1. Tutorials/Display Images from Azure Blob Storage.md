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

## References

This tutorial is based on the following articles

Working with Azure Blob Storage:

https://learn.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-dotnet?tabs=net-cli%2Cmanaged-identity%2Croles-azure-portal%2Csign-in-azure-cli%2Cidentity-visual-studio&pivots=blob-storage-quickstart-scratch

Lifetime:

https://devblogs.microsoft.com/azure-sdk/lifetime-management-and-thread-safety-guarantees-of-azure-sdk-net-clients/


# Happy Developing! 🚀




## Working with Azure Storage Accounts Locally

Developing cloud applications often involves interacting with resources in the cloud itself. However, constantly interacting with live cloud resources during development can be slow and cost-ineffective. 

This tutorial guides you through setting up your local development environment for working with Azure Storage accounts using two tools allowing you to emulate Azure Storage accounts locally:

- Azurite emulator (VSCode Extension)
- Azure Storage Explorer (Application for Windows, Mac and Linux)

### Using the Azurite Emulator with VSCode

Azurite is an open-source storage emulator supported by Azure. It emulates Azure Blob, Queue, and Table storage services locally, enabling offline development. The easiest way to use Azurite is through its VSCode extension.

#### Setting up Azurite in VSCode:

1. **Install the Azurite Extension**: Open VSCode, go to the Extensions view by clicking on the extension icon on the sidebar. Search for "Azurite" and install the extension.
2. **Start Azurite**: Once installed, open the Command Palette with `Ctrl+Shift+P` (or `Cmd+Shift+P` on Mac), type "Azurite: Start", and select the command. This action starts the emulator and creates a default storage account locally. (You will also find the Azurite services in the bottom status bar, where you can easily toggle the services on and off)

#### Configuring Your Application:


1. Create the Blob storage container in the emulator
	- Go to the "Azure" icon in the VSCode left menu bar
	- Under the "Workspace" section you find the emulator. Expand "Attached Storage Accounts"
	- Right click the Blob Containers and create the `imagerepository`


2. To interact with the emulated storage account, update your application's storage connection string to use Azurite's default settings in the Development environment:

	> appsettings.Development.json

	```json
	  "AzureBlobImageService": {
	    "ConnectionString": "DefaultEndpointsProtocol=http;AccountName=devstoreaccount1;AccountKey=Eby8vdM02xNOcqFlqUwJPLlmEtlCDXJ1OUzFT50uSRZ6IFsuFq2UVErCz4I6tq/K1SZFPTOtr/KBHBeksoGMGw==;BlobEndpoint=http://127.0.0.1:10000/devstoreaccount1;",
	    "ContainerName": "imagerepository"
	  }
	```


For guidance on Azurite and detailed setup instructions, go to the [official Microsoft Learn documentation](https://learn.microsoft.com/en-us/azure/storage/common/storage-use-azurite?tabs=visual-studio-code%2Cblob-storage#connect-to-azurite-with-sdks-and-tools).

### Exploring Local Storage with Azure Storage Explorer

Azure Storage Explorer is a standalone application that allows you to manage Azure Storage data. It supports browsing data in Azure Blob, File Shares, Queues, Tables, and Cosmos DB and it also **works with local data emulated by Azurite**.

#### Setting Up Azure Storage Explorer:

1. **Download Azure Storage Explorer**: Go to the [Azure Storage Explorer download page](https://azure.microsoft.com/en-us/products/storage/storage-explorer) and download the version compatible with your operating system.
2. **Install and Open Azure Storage Explorer**: Follow the installation guide. Once installed, launch the application.

#### Connecting to Azurite:

1. **Open Azure Storage Explorer** and navigate to the "Local & Attached" section in the Explorer pane.
2. **Connect to Azurite**: Right-click on "Storage Accounts", select "Connect to Azure Storage...", and choose "Attach to a local emulator (standard ports)".
3. **Replicate Cloud Settings Locally**: Right-click on "imagerepository" to change the settings for *public access settings* to Blob
4. Upload some content

If you want to know more about Azure Storage Explorer, go to the [official documentation](https://docs.microsoft.com/azure/vs-azure-tools-storage-manage-with-storage-explorer).

> You can run **Azurite: Clean** from the Command Palette do delete the local blob storage.

##### Verify

Start you application and verify that you see the local content

### Conclusion

Azurite, combined with the Azure Storage Explorer, provides a great setup for local development, allowing you to emulate, manage, and interact with storage accounts locally.




























