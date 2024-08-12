+++
title = 'ASP.NET MVC WebApp'
weight = 1
date = 2024-08-12
draft = false
+++

## The MVC Pattern

### Create a new MVC project

```bash
dotnet new mvc -o MVCDemo
dotnet run
```


### Controller

1. Create a new controller - DemoController. Add an action `Index()` that returns a string

	> `/Controllers/DemoController.cs`
	
	```csharp
	using Microsoft.AspNetCore.Mvc;
	using System.Text.Encodings.Web;
	
	namespace MyMvcApp.Controllers;
	
	public class DemoController : Controller
	{
	    public string Index()
	    {
	        return "This is my action...";
	    }
	}
	```

1. Add an action `HelloWorld()` without parameters that returns HtmlEncodeed string. 
	- Show the path and explain how **routing** works (*/[Controller]/[ActionName]*)
	- Explain the naming convention between rout and controller: __Demo__Controller
	- Explain XSS

	```csharp
	public string HelloWorld()
	{
	    return HtmlEncoder.Default.Encode("Hello World!");
	}
	```

1. Add an action `Hello()` that responds to a query string
	- Show with empty query string (/demo/hello)
	- Show with query string (/demo/hello?name=John&id=123)
	- Show how the id is bound to the path (/demo/hello/34?name=John)

	```csharp
	public string Hello(string name, string ID = "1")
	{
	    return HtmlEncoder.Default.Encode($"Hello {name}, {ID}!");
	}
	```
	- Explain _/[Controller]/[ActionName]/**[Parameters]**_ in the app.MapControllerRoute pattern

	> `Program.cs`
	
	```csharp
	app.MapControllerRoute(
	    name: "default",
	    pattern: "{controller=Home}/{action=Index}/{id?}");
	```
	
	- Show how you can return an `IActionResult` instead (/demo/goodbye?name=Lasse)

	```csharp
    public IActionResult Goodbye (string name)
    {
        var message = $"Goodbye {name}!";
        return Content(message, "text/html");
    }
	```
	
	
### View

1. Create a new view file `DemoAction.cshtml` and a new action `DemoAction()` in the DemoController
	- Explain the naming convention between the controller/action and the view:
		- __Demo__Controller - _DemoAction()_ ---> /Views/__Demo__/_DemoAction_.cshtml

	> `/Controllers/DemoController.cs`
	
	```csharp
    public IActionResult DemoAction ()
    {
        return View();
    }
	```
	
	> `/Views/Demo/DemoAction.cs`
	
	```html
	<h1>DemoAction</h1>
	```
	
	- Show the /Shared/Layout.cshtml file. Point out:
		- `@RenderBody()` - Razor method that renders the main view
		- `<li class="nav-item">` - Top menu item
	
1. Show how data can be transferred from the controller to the view with ViewData[]
	- Explain how the view uses the Razor template engine to render html. Note the @ for razor syntax.

	> `/Controllers/DemoController.cs`
	
	```csharp
    public IActionResult DemoAction ()
    {
    	ViewData["Message"] = "This is a demo action.";
       return View();
    }
	```
	
	> `/Views/Demo/DemoAction.cshtml`
	
	```html
	<h1>DemoAction</h1>
	<p>@ViewData["Message"]</p>
	```
	
1. Introduce forms to show how data can be sent from the view (in the browser) to the controller. Add a new action for the POST method and with a signature to recive the data from the form input field. 
	- Use plain HTML

	> `/Controllers/DemoController.cs`
	
	```csharp
    [HttpPost]
    public IActionResult DemoAction(string name)
    {
        ViewData["Name"] = $"Hello {name}!";
        return View();
    }
	```
	
	> `/Views/Demo/DemoAction.cshtml`
	
	```html
	<h1>DemoAction</h1>
	<p>@ViewData["Message"]</p>
	
	<!-- Plain HTML form with a POST method. The form will be submitted to the same page, and the data will be processed by the controller. -->
	<form method="post">
	    <input type="text" name="name" />
	    <input type="submit" value="Submit" />
	</form>
	
	<!-- Show the view data. -->
	<p>@ViewData["Name"]</p>
	```

	- Use ASP helper tags
	
	> `/Views/Demo/DemoAction.cshtml`
	
	```html
	<h1>DemoAction</h1>
	<p>@ViewData["Message"]</p>
	
	<!-- Plain HTML form with a POST method. The form will be submitted to the same page, and the data will be processed by the controller. -->
	<p>Plain HTML form</p>
	<form method="post">
	    <input type="text" name="name" />
	    <input type="submit" value="Submit" />
	</form>
	
	<!-- Use asp.net tag helpers to create a form. The form will be submitted to the same page, and the data will be processed by the controller. -->
	<p>ASP Tag helper form</p>
	<form asp-controller="Demo" asp-action="DemoAction" method="post">
	    <input type="text" name="name" />
	    <input type="submit" value="Submit" />
	</form>
	
	<!-- Show the view data. -->
	<p>@ViewData["Name"]</p>
	```

### Model

1.	Create a new model file `Person.cs` in the /Models directory

	> `/Models/Person.cs`
	
	```csharp
	namespace MyMvcApp.Models;
	
	public class Person
	{
	    public string? Name { get; set; }
	    public int Age { get; set; }
	}
	```
	
	- Add a new action in the controller. (Don´t forget the `using MyMvcApp.Models;` statement)

	> `/Controllers/DemoController.cs`
	
	```csharp
	using MyMvcApp.Models;
	...
		
	public IActionResult PersonInfo()
	{
	    var person = new Person
	    {
	        Name = "John",
	        Age = 42
	    };
	    return View(person);
	}
	```
	
	- Add a new view to show the model data

	> `/Views/Demo/PersonInfo.cshtml`
	
	```html
	@model MyMvcApp.Models.Person
	
	<h1>Person Info</h1>
	<p>@Model.Name is @Model.Age years old</p>
	```
1. Add a form to the view that uses model binding to interact with the model

	> `/Views/Demo/PersonInfo.cshtml`
	
	```html
	@model MyMvcApp.Models.Person
	
	<h1>Person Info</h1>
	
	<form asp-action="PersonInfo" method="post">
	    <div class="form-group">
	        <label for="Name">Name:</label>
	        <input type="text" class="form-control" id="Name" asp-for="Name" required />
	        <span asp-validation-for="Name" class="text-danger"></span>
	    </div>
	    <div class="form-group">
	        <label for="Age">Age:</label>
	        <input type="number" class="form-control" id="Age" asp-for="Age" required />
	        <span asp-validation-for="Age" class="text-danger"></span>
	    </div>
	    <button type="submit" class="btn btn-primary">Submit</button>
	</form>
	
	<p>@Model.Name is @Model.Age years old</p>
	```

	- Add a new action that listens to the POST request

	> `/Controllers/DemoController.cs`
	
	```csharp
    [HttpPost]
    public IActionResult PersonInfo(Person model)
    {
        if (ModelState.IsValid)
        {
            // Save to database or any other logic here
            //return RedirectToAction("Success"); // Redirect to a success page
            return View(model);
        }

        // If the model is not valid, return the same view to display errors
        return View(model);
    }
	```
	- Add validation to the input fields
		- Decorate the model
		- Add info script to the view

	> `/Models/Person.cs`
	
	```csharp
	using System.ComponentModel.DataAnnotations;
	
	namespace MyMvcApp.Models;
	
	public class Person
	{
	    [Required]
	    [RegularExpression(@"^[a-zA-Z]+$", ErrorMessage = "Use letters only, please")]
	    public string Name { get; set; }
	
	    //public DateTime Born { get; set; }
	    public int Age { get; set; }
	}
	```

	> `/Views/Demo/PersonInfo.cshtml`
	
	```html
	@model MyMvcApp.Models.Person
	
	<h1>Person Info</h1>
	
	<form asp-action="PersonInfo" method="post">
	    <div class="form-group">
	        <label for="Name">Name:</label>
	        <input type="text" class="form-control" id="Name" asp-for="Name" required />
	        <span asp-validation-for="Name" class="text-danger"></span>
	    </div>
	    <div class="form-group">
	        <label for="Age">Age:</label>
	        <input type="number" class="form-control" id="Age" asp-for="Age" required />
	        <span asp-validation-for="Age" class="text-danger"></span>
	    </div>
	    <button type="submit" class="btn btn-primary">Submit</button>
	</form>
	
	<p>@Model.Name is @Model.Age years old</p>
	
	@section Scripts {
	    @{await Html.RenderPartialAsync("_ValidationScriptsPartial");}
	}
	```

## Add A Service

In this section we will introduce:

- DTO (Data Transfer Object)
- Interface
- Application Service
- Dependency Injection


1. Add a DTO to transfer data from the application layer to the presentation layer

	- Add a new directory `Application` and a new file `PersonDTO.cs`

	> `/Application/PersonDTO.cs`
	
	```csharp
	namespace MVCDemo.ApplicationServices;
	
	public class PersonDTO
	{
	    public string? Name { get; set; }
	    public int Age { get; set; }
	}
	```

1. Add an application service interface

	- Add a new file `IPersonService.cs`

	> `/Application/IPersonService.cs`
	
	```csharp
	namespace MVCDemo.ApplicationServices;
	
	public interface IPersonService
	{
	    PersonDTO GetPerson();
	}
	```

1. Add an application service implementation

	- Add a new file `PersonService.cs`

	> `/Application/PersonService.cs`
	
	```csharp
	namespace MVCDemo.ApplicationServices;
	
	public class PersonService : IPersonService
	{
	    public PersonDTO GetPerson()
	    {
	        // Return a mock PersonDTO object
	        return new PersonDTO
	        {
	            Name = "JohnDoe",
	            Age = 44
	        };
	    }
	}
	```

1. Add the application service to dependency injection

	_Scoped_ means that the object will live through an entire HTTP request and response cycle

	> `/Program.cs`
	
	```csharp
	using MVCDemo.ApplicationServices;
	
	...
	
	builder.Services.AddScoped<IPersonService, PersonService>();
	
	...
	
	```

1. Change the controller to use the service. Use constructor injection to start using the service

	> `/Controllers/DemoController.cs`
	
	```csharp
	using MVCDemo.ApplicationServices;
	
	...
	// Inject the service through the ctor
	
	public class DemoController : Controller
	{
	    private readonly IPersonService _personService;
	
	    public DemoController(IPersonService personService)
	    {
	        _personService = personService;
	    }
	
	...
	
	// Change the PersonInfo method to use the service and the DTO to retreive person info
	public IActionResult PersonInfo()
	{
	    // Get the person from the service
	    var personDTO = _personService.GetPerson();
	    var person = new Person
	    {
	        Name = personDTO.Name,
	        Age = personDTO.Age
	    };
	
	    // Comment out the old code
	    // var person = new Person
	    // {
	    //     Name = "John",
	    //     Age = 42
	    // };
	
	    return View(person);
	}

	```

## The Domain Layer

The domain layer is the core of the system. It has no dependencies to other parts of the application. It typically defines Domain Entities and Interfaces to the infrastructure layer.

1. Add a new folder called `Domain` and a new file iin that folder called `Person.cs`. This person class defines the domain entity _Person_. We will later adhere to stricter encapsulation and add validation to this class. But for now we keep it simple.

	> `/Domain/Person.cs`
	
	```csharp
	namespace MVCDemo.Domain;
	
	public class Person
	{
	    public string Name { get; set; }
	    public int Age { get; set; }
	
	    public Person(string name, int age)
	    {
	        Name = name;
	        Age = age;
	    }
	}	
	```

1. Next we will use the Repository Pattern to remove the dependency to the data storage implementation. The interface, or the contract, to store the data is however a responsability of the Domain Layer. It is often defined in a CRUD like manor, but we keep it simple here and have only one Get method.

	- Add a new file `IPersonRepository.cs`

	> `/Domain/IPersonRepository.cs `
	
	```csharp
	namespace MVCDemo.Domain;
	
	public interface IPersonRepository
	{
	    Person GetPersonById(int id);
	}
	```
	
## The Infrastructure Layer

As mentioned before is it the responsability of the Domain Layer to define the Interface. It is however the responsability of the Infrastructure Layer to implement the class inheriting from the interface. This is what makes the Domain Layer independent. 

1. Add a new folder called `Infrastructure` and a new file in that folder called `PersonRepository.cs `. 

	> `/Infrastructure/PersonRepository.cs `
	
	```csharp
	namespace MVCDemo.Infrastructure;
	
	using MVCDemo.Domain;
	
	public class PersonRepository : IPersonRepository
	{
	    public Person GetPersonById(int id)
	    {
	        return new Person("John", 42); // Mock implementation
	    }
	}
	```

1. Next we want to register the repository interface and its implementation in the DI Container.

	- Add the repository to dependency injection

	> `/Program.cs`
	
	```csharp
	using MVCDemo.Domain;
	using MVCDemo.Infrastructure;
	
	...
	
	builder.Services.AddScoped<IPersonService, PersonService>();
	builder.Services.AddScoped<IPersonRepository, PersonRepository>();
	...
	
	```

1. Let us now return to the Application Layer and the _PersonService_ and use the repository to retreive the data. Note that the application layer will depend on the Domain but __not__ on the Infrastructure Layer, since the repository interface is defined in the Domain Layer.

	- Update the PersonService. We get the repository from the DI Container via the constructor

	> `/Application/PersonService.cs`
	
	```csharp
	namespace MVCDemo.ApplicationServices;
	using MVCDemo.Domain;
	
	public class PersonService : IPersonService
	{
	    private readonly IPersonRepository _personRepository;
	
	    public PersonService(IPersonRepository personRepository)
	    {
	        _personRepository = personRepository;
	    }
	
	    public PersonDTO GetPerson()
	    {
	        // Get a Person object from the repository
	        var person = _personRepository.GetPersonById(1);
	        return new PersonDTO
	        {
	            Name = person.Name,
	            Age = person.Age
	        };
	
	        // // Return a mock PersonDTO object
	        // return new PersonDTO
	        // {
	        //     Name = "JohnDoe",
	        //     Age = 44
	        // };
	    }
	}
	```

### Swap the repository implementation

The repository pattern enables us to rather easily swap one repository implementation for another. Previously we used an in-memory structuree to mock the data. In this chapter we will implement and use a different repository that reads the data from a json file instead.

1. First let us create the json file with data. Note that it is common to use `snake_case`in JSON.

	> `/persons.json`
	
	```json
	[
	    {
	        "id": 1,
	        "name": "John",
	        "age": 42
	    },
	    {
	        "id": 2,
	        "name": "Jane",
	        "age": 36
	    }
	]
	```
	
1. From the repository methods we return domain entity objects. However, in order to handle the data within the repository implementation we will introduce a data entity object that is taylored for the actual persistence method we have chosen, JSON in our case.

	- Add a new file in the infrastructure folder called `PersonEntityJson.cs`. Note that it uses decorators to map the `snake_case` naming convention in JSON to the `PascalCase` naming convention used in C#.

	
	> `/Infrastructure/PersonEntityJson.cs `
	
	```csharp
	using System.Text.Json.Serialization;
	
	namespace MVCDemo.Infrastructure;
	public class PersonEntityJson
	{
	    [JsonPropertyName("id")]
	    public int Id { get; set; }
	
	    [JsonPropertyName("name")]
	    public string? Name { get; set; }
	
	    [JsonPropertyName("age")]
	    public int Age { get; set; }
	}
	```
	
1. Write a new repository implementation. Add a new file in the infrastructure folder called `PersonRepositoryJson.cs`.

	
	> `/Infrastructure/PersonRepositoryJson.cs `
	
	```csharp
	using System.Text.Json;
	
	using MVCDemo.Domain;
	
	namespace MVCDemo.Infrastructure;
	
	public class PersonRepositoryJson : IPersonRepository
	{
	    private readonly string _filePath = "persons.json";
	
	    public Person GetPersonById(int id)
	    {
	        var persons = ReadFromFile();
	        var personData = persons.Find(p => p.Id == id);
	
	        return personData != null ? new Person(personData.Name, personData.Age) : null;
	    }
	
	    private List<PersonEntityJson> ReadFromFile()
	    {
	        if (File.Exists(_filePath))
	        {
	            var json = File.ReadAllText(_filePath);
	            return JsonSerializer.Deserialize<List<PersonEntityJson>>(json);
	        }
	
	        return new List<PersonEntityJson>();
	    }
	}
	```

4. Now we need to swap the implementation in the DI Container, which we do in `Program.cs`.

	- Comment out the previous row and add the new implementation


	> `/Program.cs`
	
	```csharp
	...
	//builder.Services.AddScoped<IPersonRepository, PersonRepository>();
	builder.Services.AddScoped<IPersonRepository, PersonRepositoryJson>();
	...
	
	```
	
	
Change the values in the JSON file and refresh the page. to verify that the new data now come from the file. 

Even if it might seem as some overhead to create the different layers and model classes it also shows how easy modules can be replaced with a new implementation withoout affecting other layers.
