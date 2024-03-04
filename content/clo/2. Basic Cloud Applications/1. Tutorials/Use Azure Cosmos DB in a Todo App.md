+++
title = "Use Azure Cosmos DB in a Todo App"
weight = 5
date = 2024-03-04
draft = false
+++

## Introduction

In this guide, we'll walk you through integrating Azure Cosmos DB into an ASP.NET Razor Pages web application to manage a simple to-do list. 

Azure Cosmos DB offers a globally distributed, multi-model database service that scales seamlessly with your needs. 

This tutorial covers setting up a Cosmos DB account configured for MongoDB API, creating a to-do list web application, and implementing CRUD operations using the Cosmos DB service. 

By the end of this guide, you'll have a functional to-do list application powered by Azure Cosmos DB, demonstrating the database's capabilities within a web development context.

## Method

- We'll start by setting up an Azure Cosmos DB account (free tier) through the Azure portal
- We will utilize MongoDB's client library (the MongoDB.Driver NuGet package) in our .NET application for database operations.
- Next, we'll create a new ASP.NET Razor Pages web application. This web application will serve as the interface for our to-do list, allowing users to create, view, update, and delete to-do items.
- Within the web application, we'll develop a to-do service that acts as a bridge between our application and Azure Cosmos DB. This service will handle all interactions with the database, such as retrieving, adding, updating, and deleting to-do items. 
- The configurations, such as the connection string, database and collection name, will be loaded into our web application’s settings via environment variables.

## Prerequisites

- An Azure account. If you don't have one, sign up at [Azure's official site](https://azure.microsoft.com/).
- Basic familiarity with Azure services and ASP.NET web development.

## Step 1: Set Up Azure Cosmos DB

1. Create a Resource Group `CosmosDemoRG`
2. Create an Azure Cosmos DB:
	- Select **Azure Cosmos DB** in the resource menu
	- Click the **+ Create** button 
		- Find _Azure Cosmos DB for MongoDB_ and press **Create**
		- Find _Request unit (RU) database account_ and press **Create**
		- Resource Group: `CosmosDemoRG`
		- Account name: `cosmosdemo<datetime>` (change datetime to the current date and time)
		- Press **Review + create**
		- Press **Create**
		- Press **Go to resource**
	- Select **Configuration** in the _Settings_ section of the storage account menu
		- (Alt: Select **Security** from storage account _Overview_)
	- Enable **Allow Blob anonymous access**
	- Press **Save**

### Retrieve the Connection String

- Select your newly created Cosmos DB `cosmosdemo<datetime>`
- Go to **Connection strings** in the _Settings_ section of the cosmos account menu
- Find _PRIMARY CONNECTION STRING_ and press **Show**. Copy the connection string and use it in your app

## Step 2: Create the Web Application

### Set Up the Webapp

Create a directory called `CosmoDB` and open up a terminal. Run the command below to create a Razor Pages webapp:

```bash
dotnet new webapp
```

Verify:

```bash
dotnet run
```

### Develop A Todo Service

#### Prepare the appsettings

> appsettings.json

```json
...

  "AllowedHosts": "*",
  "AzureCosmosDBTodoService": {
    "ConnectionString": "Set ConnectionString in environment variable or user secrets",
    "Database": "Set ConnectionString in environment variable or user secrets",
    "Collection": "Set Collection in environment variable or user secrets"
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

AzureCosmosDBTodoService__ConnectionString="Paste in the connection string here"
AzureCosmosDBTodoService__Database="Paste in the container name here"
AzureCosmosDBTodoService__Collection="Paste in the container name here"

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
dotnet add package MongoDB.Driver
```

#### Define a Todo Model

Create a new directory `Models`.

Create a new file in the `Models` directory called `TodoItem.cs`

> /Models/TodoItem.cs

```csharp
using System.ComponentModel.DataAnnotations;
using MongoDB.Bson;
using MongoDB.Bson.Serialization.Attributes;

namespace CosmosDB.Models;

public class TodoItem
{
    [BsonId]
    [BsonRepresentation(BsonType.String)]
    public Guid Id { get; set; } = Guid.NewGuid(); // Auto-generate GUID
    [Required(ErrorMessage = "The Title field is required")]
    public string? Title { get; set; }
    public bool IsComplete { get; set; }
}
```

> The decorators (attributes in C#) used in the `TodoItem` class are annotations that provide metadata about the properties they decorate, influencing how these properties are handled by MongoDB's .NET driver during serialization and deserialization processes. Here's a brief explanation of each:
> 
> **`[BsonId]`**
> 
> - **Purpose**: Marks the property as the document's primary key in MongoDB.
> - **Effect**: The property it decorates will map to the `_id` field in a MongoDB document, which is a unique identifier for the document in its collection.
> 
> **`[BsonRepresentation(BsonType.String)]`**
> 
> - **Purpose**: Specifies how a property's value should be represented in BSON when the document is stored in MongoDB.
> - **Effect**: Despite the property being a `Guid` in C#, it will be stored as a string in MongoDB. This is useful for ensuring the `Guid` values are readable and compatible with systems that might not recognize `Guid` data types directly but can work with strings.

#### Develop a Service Interface

Create a new directory `Services`.

Create a new file in the `Services` directory called `ITodoService.cs`

> /Services/ITodoService.cs

```csharp
using CosmosDB.Models;

namespace CosmosDB.Services;

public interface ITodoService
{
    Task<IEnumerable<TodoItem>> GetAllAsync();
    Task<TodoItem> GetByIdAsync(Guid id);
    Task<TodoItem> CreateAsync(TodoItem item);
    Task<TodoItem> UpdateAsync(Guid id, TodoItem item);
    Task<TodoItem> DeleteAsync(Guid id);
}
```


#### Develop a service implementation using Azure Cosmos DB

Create a new file in the `Services` directory called `AzureCosmosDBTodoService.cs`

> /Services/AzureCosmosDBTodoService.cs

```csharp
using MongoDB.Driver;
using CosmosDB.Models;

namespace CosmosDB.Services;

public class AzureCosmosDBTodoService : ITodoService
{
    private readonly IMongoCollection<TodoItem> _todoItems;

    public AzureCosmosDBTodoService(IConfiguration configuration)
    {
        // Get the configuration settings from dependency injection
        var cosmosDbServiceConfig = configuration.GetSection("AzureCosmosDBTodoService");
        string connectionString = cosmosDbServiceConfig["ConnectionString"] ?? throw new InvalidOperationException("ConnectionString");
        string databaseName = cosmosDbServiceConfig["Database"] ?? throw new InvalidOperationException("Database");
        string collectionName = cosmosDbServiceConfig["Collection"] ?? throw new InvalidOperationException("Collection");

        // Create a MongoClient object by passing the connection string
        var client = new MongoClient(connectionString);

        // Get the database (creates if it doesn't exist)
        var database = client.GetDatabase(databaseName);

        // Get the collection (creates if it doesn't exist)
        _todoItems = database.GetCollection<TodoItem>(collectionName);
    }

    public async Task<IEnumerable<TodoItem>> GetAllAsync()
    {
        return await _todoItems.Find(item => true).ToListAsync();
    }

    public async Task<TodoItem> GetByIdAsync(Guid id)
    {
        return await _todoItems.Find(item => item.Id == id).FirstOrDefaultAsync();
    }

    public async Task<TodoItem> CreateAsync(TodoItem item)
    {        
        // A new ID is generated if not provided. See the TodoItem class.
        await _todoItems.InsertOneAsync(item);
        return item; // After insertion, the item will have the Id set by MongoDB.
    }

    public async Task<TodoItem> UpdateAsync(Guid id, TodoItem item)
    {
        await _todoItems.ReplaceOneAsync(t => t.Id == id, item);
        return item;
    }

    public async Task<TodoItem> DeleteAsync(Guid id)
    {
        return await _todoItems.FindOneAndDeleteAsync(item => item.Id == id);
    }
}
```

#### Register the service in Dependeny Injection

> Program.cs

```csharp
using CosmosDB.Services;

...

// Register the IToDoService with the DI container
builder.Services.AddSingleton<ITodoService, AzureCosmosDBTodoService>();

...

app.Run();
```

## Present the images on the web page

### Create an OnGetAsync method in the controller

> /Pages/Index.cshtml.cs

```csharp
using CosmosDB.Models;
using CosmosDB.Services;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.RazorPages;

namespace CosmosDB.Pages;

public class IndexModel : PageModel
{
    private readonly ILogger<IndexModel> _logger;
    private readonly ITodoService _todoService;
    public IEnumerable<TodoItem> TodoItems { get; set; } = [];

    // Model binding for the "Add Todo" form
    [BindProperty]
    public TodoItem NewTodo { get; set; } = new();

    public IndexModel(ILogger<IndexModel> logger, ITodoService todoService)
    {
        _logger = logger;
        _todoService = todoService;
    }

    public async Task<IActionResult> OnGetAsync()
    {
        // Use the ITodoService to get all the TodoItems
        TodoItems = await _todoService.GetAllAsync();

        // Return the Page with the list of TodoItems
        return Page();
    }

    public async Task<IActionResult> OnPostAddTodoAsync()
    {
        if (!ModelState.IsValid)
        {
            return Page();
        }

        // Use the ITodoService to create a new TodoItem
        await _todoService.CreateAsync(NewTodo);
        
        // Redirect to the same page to refresh the list of TodoItems
        return RedirectToPage();
    }

    public async Task<IActionResult> OnPostToggleTodoIsCompleteAsync(Guid Id)
    {
        // Use the ITodoService to get the TodoItem by Id
        var todo = await _todoService.GetByIdAsync(Id);

        // Toggle the IsComplete property of the TodoItem
        todo.IsComplete = !todo.IsComplete;

        // Use the ITodoService to update the TodoItem
        await _todoService.UpdateAsync(Id, todo);

        // Redirect to the same page to refresh the list of TodoItems
        return RedirectToPage();
    }

    public async Task<IActionResult> OnPostDeleteCompletedTodosAsync()
    {
        // Use the ITodoService to get all the TodoItems
        var todoItems = await _todoService.GetAllAsync();

        // Use LINQ to get all the completed TodoItems
        var todosToDelete = todoItems.Where(t => t.IsComplete == true);
        foreach (var todo in todosToDelete)
        {
            // Use the ITodoService to delete all the completed TodoItems
            await _todoService.DeleteAsync(todo.Id);
        }

        // Redirect to the same page to refresh the list of TodoItems
        return RedirectToPage();
    }

}
```

### Update the page to list the images

> /Pages/Index.cshtml

```html
@page
@model IndexModel
@{
    ViewData["Title"] = "Home page";
}

<h3>Add ToDo</h3>

<form method="post" asp-page-handler="AddTodo"> 
    <input type="text" asp-for="NewTodo.Title" placeholder="Add a new ToDo" />
    <button type="submit">Add</button> 
    <span asp-validation-for="NewTodo.Title" class="text-danger"   ></span>
</form>

<h3>Delete All Completed</h3>

<form method="post" asp-page-handler="DeleteCompletedTodos">
    <button type="submit">Delete Completed</button>
</form>

<h3>ToDo List</h3>

<ul>
    @foreach (var todoItem in Model.TodoItems)
    {
        <form method="post" asp-page-handler="ToggleTodoIsComplete">
        <input type="hidden" name="id" value="@todoItem.Id" />
        <input type="checkbox" asp-for="@todoItem.IsComplete" onchange="this.form.submit();"/>
        @todoItem.Title
        </form>
    }
</ul>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

### Style the page with Bootstrap

Create a new file in the `Pages` directory called `Index.cshtml.css`

> /Pages/Index.cshtml.css

```css
/* Scoped styles for Index.cshtml */

.todo-item.done {
    text-decoration: line-through; /* Strikethrough */
    color: gray; /* gray it out */
}
```

Change the `Index.cshtml` file to the following:

> /Pages/Index.cshtml

```html
@page
@model IndexModel
@{
    ViewData["Title"] = "Home page";
}

<div class="container mt-5">

    <!-- Add ToDo Card -->
    <div class="card mb-3" >
        <div class="card-header text-white bg-success">
            <h3 class="card-title mb-1 mt-1">Add ToDo</h3>
        </div>
        <div class="card-body">
            <form method="post" asp-page-handler="AddTodo">
                <input type="text" asp-for="NewTodo.Title" class="form-control" placeholder="Add a new ToDo"/>
                <button type="submit" class="btn btn-primary mt-2">Add</button>
                <span asp-validation-for="NewTodo.Title" class="text-danger"></span>
            </form>
        </div>
    </div>

    <!-- ToDo List Card -->
    <div class="card bg-success">
        <div class="card-header d-flex justify-content-between align-items-center text-white">
            <h3 class="card-title mb-1 mt-1">ToDo List</h3>
            <form method="post" asp-page-handler="DeleteCompletedTodos">
                <button type="submit" class="btn btn-danger">Delete All Completed</button>
            </form>
        </div>
        <ul class="list-group list-group-flush">
            @foreach (var todoItem in Model.TodoItems)
            {
                <div class="list-group-item">
                    <form method="post" asp-page-handler="ToggleTodoIsComplete">
                        <input type="hidden" name="id" value="@todoItem.Id" />
                        <div class="form-check">
                            <input type="checkbox" class="form-check-input" asp-for="@todoItem.IsComplete" id="@todoItem.Id" onchange="this.form.submit();"/>
                            <label class="form-check-label todo-item @(todoItem.IsComplete ? "done" : "")" for="@todoItem.Id">
                                @todoItem.Title
                            </label>
                        </div>
                    </form>
                </div>
            }
        </ul>
    </div>
</div>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```


### Run the webapp

Verify that you can see a list of images:

```bash
dotnet run
```

## Conclusion

This tutorial has walked you through the process of integrating Azure Cosmos DB, utilizing the MongoDB API, into an ASP.NET Razor Pages web application to manage a to-do list. By setting up Azure Cosmos DB, creating the web application, and developing the necessary services and models, you've learned how to perform CRUD operations on a Cosmos DB database from within an ASP.NET application.

## Don't Forget

Azure services incur costs. Delete resources you no longer need.

# Happy Developing! 🚀
