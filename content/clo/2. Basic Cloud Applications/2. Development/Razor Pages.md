+++
title = "Understanding Razor Pages in ASP.NET Core"
weight = 1
date = 2024-03-03
draft = false
+++

## Introduction

Razor Pages is a framework for building web applications in ASP.NET Core, aimed at simplifying the web development process. It is designed to offer an efficient way to build dynamic web pages, integrating closely with the MVC pattern but with a focus on page-centric development. This guide describes the key concepts of Razor Pages.

Razor Pages streamlines web UI development by combining the model and controller layers into a single _Page Model_ for each web page. This structure facilitates direct development paths for creating web applications, making it accessible for developers to work with complex server-side logic in a straightforward manner.

## Core Concepts

The foundation of Razor Pages rests on several concepts central to its operation and effectiveness in web development. Unlike MVC, which separates an application into models, views, and controllers, Razor Pages combines the model and controller aspects into a single Page Model. 

### Razor Syntax

Razor syntax is a mix of C# and HTML, allowing for dynamic HTML content generation server-side. It's designed for both productivity and readability, enabling developers to embed server-side logic directly within HTML markup.

### Tag Helpers

Tag Helpers enhance HTML by adding server-side attributes to tags, simplifying the incorporation of server-side logic into views. They make HTML generation more intuitive, connecting front-end and back-end development more seamlessly. Tag Helpers are extensively used in Razor Pages for form submissions, validation, linking to other pages, and more.

### Page Model

Acting as a fusion of the model and controller, the Page Model contains the server-side logic for a page, including data handling and response generation. It separates the business logic from the presentation layer, improving maintainability and testability.

The Page Model approach, which is similar to the *ViewModel* pattern. Each Razor Page (.cshtml file) has a corresponding Page Model class (.cshtml.cs) that holds the request handling logic, data, and any actions related to that page. 

### Routing

Razor Pages uses a *convention-based routing* system, automatically mapping URLs to Razor files based on their project directory location. This eliminates the need for complex routing configurations and focuses development efforts on functionality.

Additionally, Page Handlers (`OnGet`, `OnPost`, etc.) are used within Page Models to handle different HTTP methods, making it easy to manage the flow of data for forms and actions within a single page.

### Model Binding and Validation

Model Binding automates the mapping of HTTP request data to Page Model properties, streamlining form submissions and data handling. Validation mechanisms ensure data integrity, utilizing Data Annotations and validation Tag Helpers for immediate user feedback.

### Dependency Injection (DI)

DI in Razor Pages promotes loose coupling, allowing for modular applications by injecting services and components directly into Page Models. This facilitates easier testing and maintenance by enabling dependency replacement or mocking.

Razor Pages applications heavily utilize Dependency Injection (DI) for accessing services and functionality, such as database contexts, logging, and business logic components. ASP.NET Core's built-in DI container is used to inject dependencies directly into the Page Model.

### Security

Razor Pages integrates ASP.NET Core's security features, including authentication, authorization, and protections against common vulnerabilities. This ensures that developers can more easily secure their web applications.

### Layouts, Partial Views, and View Components

While not exclusive to Razor Pages, the use of layouts, partial views, and view components is common in Razor Pages applications for code reuse and organization. These elements help in maintaining a consistent look and feel across the web application while avoiding duplication of markup.

### Asynchronous Programming

Razor Pages supports asynchronous programming, allowing for non-blocking operations which are essential for performing tasks such as data access, file I/O, or network requests efficiently. This capability is critical in web development for enhancing application performance and scalability. By leveraging the `async` and `await` keywords, Razor Pages can execute time-consuming operations in the background, freeing up the web server to handle other requests in the meantime.


## Examples

To illustrate the core concepts of Razor Pages we can use a simple todo application as an example, let's illustrate each concept with relevant code snippets.

### Razor Syntax

Razor syntax blends C# and HTML, allowing dynamic content generation. In a todo application, a Razor Page might display a list of todo items as follows:

```html
@page
@model TodoListModel

<h1>Todo List</h1>
<ul>
@foreach (var item in Model.TodoItems) {
    <li>@item.Name</li>
}
</ul>
```

Here, `@model TodoListModel` indicates the Page Model this Razor Page is connected to, and the `@foreach` loop dynamically generates HTML list items based on the todo items in the model.

### Page Model

The Page Model contains server-side logic. For our todo list, the Page Model might look like this:

```csharp
public class TodoListModel : PageModel
{
    public List<TodoItem> TodoItems { get; set; } = new List<TodoItem>();

    public void OnGet()
    {
        // Ideally, items would be fetched from a database
        TodoItems.Add(new TodoItem { Name = "Learn ASP.NET Core" });
        TodoItems.Add(new TodoItem { Name = "Build a todo app" });
    }
}
```

This model fetches todo items when the page is accessed via a GET request.

### Tag Helpers

Tag Helpers enhance HTML with server-side functionality. For adding a new todo item, a form might use Tag Helpers as shown:

```html
<form method="post">
    <input asp-for="NewItemName" />
    <button type="submit">Add</button>
</form>
```

`asp-for` binds the input to a property in the Page Model, enabling model binding when the form is submitted.

### Routing

Routing is convention-based. For a Razor Page located at `/Pages/Todos/Index.cshtml`, the URL would be `/Todos/Index`. Custom routes can be specified using the `@page` directive:

```html
@page "/todo/list"
```

This changes the page's route to `/todo/list`.

### Model Binding and Validation

Model binding maps data from HTTP requests. For adding a new todo item:

```csharp
public class TodoListModel : PageModel
{
    [BindProperty]
    public string NewItemName { get; set; }

    public void OnPost()
    {
        // Add the new item to the list
    }
}
```

Validation ensures data integrity. By annotating the `NewItemName` with data annotations, you can enforce validation rules:

```csharp
[BindProperty]
[Required]
public string NewItemName { get; set; }
```

### Dependency Injection (DI)

DI allows services to be injected into the Page Model. If the todo items are stored in a database, you might have a service for data access:

```csharp
public class TodoListModel : PageModel
{
    private readonly ITodoService _todoService;

    public TodoListModel(ITodoService todoService)
    {
        _todoService = todoService;
    }

    public void OnGet()
    {
        TodoItems = _todoService.GetItems();
    }
}
```

### Security

Razor Pages supports security features like authentication and authorization. Restricting access to a page can be as simple as adding an attribute to the Page Model:

```csharp
[Authorize]
public class TodoListModel : PageModel
{
    // Page Model content
}
```

This ensures only authenticated users can access the todo list.

### Async

A simple example demonstrating the use of asynchronous operations in a Razor Page Model, specifically for fetching todo items from a database asynchronously:

```csharp
public class TodoListModel : PageModel
{
    private readonly ITodoService _todoService;

    public TodoListModel(ITodoService todoService)
    {
        _todoService = todoService;
    }

    public List<TodoItem> TodoItems { get; private set; }

    public async Task OnGetAsync()
    {
        TodoItems = await _todoService.GetItemsAsync();
    }
}
```

In this example, `GetItemsAsync` is an asynchronous method in the `ITodoService` service that retrieves todo items from a database. The `OnGetAsync` method is marked with `async`, and it awaits the completion of `GetItemsAsync` before proceeding. This ensures that the operation doesn't block the thread, improving the overall efficiency of the application.


## Conclusion

Razor Pages is designed to make web development straightforward, focusing on productivity and maintainability. By understanding its core concepts — from Razor syntax and Page Models to DI and security — developers can use this framework to build dynamic, secure web applications.