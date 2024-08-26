+++
title = 'Web API With Clients'
weight = 4
date = 2024-08-26
draft = false
+++

## Demo: API

Create a subfoder `APIDemo.API`

```bash
dotnet new webapi --use-controllers -o APIDemo.API
dotnet run
```

### Model

1.	Create a new model directory `Models` and a new file `Product.cs` in the /Models directory

	> `/Models/Product.cs `
	
	```csharp
	namespace APIDemo.API.Models;
		
	public class Product
	{
	    public int Id { get; set; }
	    public string? Name { get; set; }
	    public string? Description { get; set; }
	    public decimal Price { get; set; }
	}
	```

### Controller

1. Create a new controller `ProductsController.cs`

	> `/Controllers/ProductsController.cs `
	
	```csharp
	using Microsoft.AspNetCore.Mvc;
	
	using APIDemo.API.Models;
	
	namespace ApiDemo.API.Controllers;
	
	/*
	 The ProductsController class follows the convention of ASP.NET Core for naming controllers and actions:
	 The controller class name ends with "Controller" -> "ProductsController".
	 The HTTP method for each action is specified using the [HttpGet], [HttpPost], [HttpPut], and [HttpDelete] attributes.
	 The name of the action methods (Get, Post, Put, Delete) is not significant. 
	 Routing is determined by [HttpGet], [HttpPost], [HttpPut], and [HttpDelete], not by the method name.
	 */
	
	[ApiController] // used to mark this class as a controller that uses the API behavior conventions
	[Route("api/[controller]")] // all actions in this controller will start with "/api/products"
	public class ProductsController : ControllerBase
	{
	    // This is a simple in-memory data store.
	    private static List<Product> products = new List<Product>
	    {
	        new Product { Id = 1, Name = "Product1", Description = "Description1", Price = 10.0M },
	        new Product { Id = 2, Name = "Product2", Description = "Description2", Price = 20.0M },
	        // ... add more products here
	    };
	
	    // Expected URL: GET /api/products
	    [HttpGet]
	    public IEnumerable<Product> Get()
	    {
	        return products;
	    }
	
	    // Expected URL: GET /api/products/{id}
	    [HttpGet("{id}")]
	    public Product? Get(int id)
	    {
	        return products.FirstOrDefault(p => p.Id == id);
	    }
	
	    // Expected URL: POST /api/products
	    [HttpPost]
	    public void Post([FromBody] Product product)
	    {
	        products.Add(product);
	    }
	
	    // Expected URL: PUT /api/products/{id}
	    [HttpPut("{id}")]
	    public void Put(int id, [FromBody] Product product)
	    {
	        var existingProduct = products.FirstOrDefault(p => p.Id == id);
	        if (existingProduct != null)
	        {
	            existingProduct.Name = product.Name;
	            existingProduct.Description = product.Description;
	            existingProduct.Price = product.Price;
	        }
	    }
	
	    // Expected URL: DELETE /api/products/{id}
	    [HttpDelete("{id}")]
	    public void Delete(int id)
	    {
	        var product = products.FirstOrDefault(p => p.Id == id);
	        if (product != null)
	        {
	            products.Remove(product);
	        }
	    }
	}
	```	

### Program

Go through `Programs.cs`

1. Setting up Controllers
2. Setting up Swagger


### Swagger

Go to `http://localhost:<port>/swagger`. Explore and try the api


## Use an ASP.NET MVC Webapp as an API Client

Create a subfoder `APIDemo.MVCClient`

```bash
dotnet new mvc
dotnet run
```

### Model

1.	Create a new model file `ProductViewModel.cs` in the /Models directory

	> `/Models/ProductViewModel.cs `
	
	```csharp
	namespace APIDemo.MVCClient.Models;
	
	/* Swagger Schema "Product"
	{
	    id  integer($int32)
	    name    string
	    nullable: true
	    description string
	    nullable: true
	    price   number($double)
	}
	*/
	
	public class ProductViewModel
	{
	    public int Id { get; set; }
	    public string? Name { get; set; }
	    public string? Description { get; set; }
	    public decimal Price { get; set; }
	}
	```

### Controller

1. Create a new controller `ProductsController.cs` and add a new action `Index()`
2. Mock the API data at this stage

	> `/Controllers/ProductsController.cs `
	
	```csharp
	using Microsoft.AspNetCore.Mvc;
	using APIDemo.MVCClient.Models;
	
	namespace APIDemo.MVCClient.Controllers;
	
	public class ProductsController : Controller
	{
	    public async Task<IActionResult> Index()
	    {
	        // Get all products from the API service
	
	        // Mocked API data
	        var products = GetProducts();
	
	        return View(products);
	    }
	
	    // Mocked API data. Private method to simulate API call
	    private List<ProductViewModel> GetProducts()
	    {
	        return new List<ProductViewModel>
	        {
	            new ProductViewModel
	            {
	                Id = 1,
	                Name = "Product 1",
	                Description = "Description for Product 1",
	                Price = 10.99m
	            },
	            new ProductViewModel
	            {
	                Id = 2,
	                Name = "Product 2",
	                Description = "Description for Product 2",
	                Price = 20.99m
	            },
	            new ProductViewModel
	            {
	                Id = 3,
	                Name = "Product 3",
	                Description = "Description for Product 3",
	                Price = 30.99m
	            }
	        };
	    }
	}
	```	
	
### View

1. Create a new directory `Products` and a new view file `Index.cshtml`
	
	> `/Views/Products/Index.cshtml `
	
	```html
	@model IEnumerable<APIDemo.MVCClient.Models.ProductViewModel>
	
	<h2>Products</h2>
	
	<table class="table">
	    <thead>
	        <tr>
	            <th>ID</th>
	            <th>Name</th>
	            <th>Description</th>
	            <th>Price</th>
	            <th></th>
	        </tr>
	    </thead>
	    <tbody>
	        @foreach (var product in Model)
	        {
	            <tr>
	                <td>@product.Id</td>
	                <td>@product.Name</td>
	                <td>@product.Description</td>
	                <td>@product.Price</td>
	                <td>
	                    <a asp-action="Edit" asp-route-id="@product.Id" class="btn btn-primary">Edit</a>
	                    <a asp-action="Delete" asp-route-id="@product.Id" class="btn btn-danger">Delete</a>
	                </td>
	            </tr>
	        }
	    </tbody>
	</table>
	
	<a asp-action="Create" class="btn btn-success">Create New</a>

	```
	

### Navigation

1.	Add a button to the menu bar in the file `_Layout.cshtml` in the `/Views/Shared` directory

	> `/Views/Shared/_Layout.cshtml `
	
	```html
    <li class="nav-item">
        <a class="nav-link text-dark" asp-area="" asp-controller="Products" asp-action="Index">Products</a>
    </li>
	```
	
### Services

1.	Create a new directory `Services`
1.	Create a new file `ProductApiDTO.cs` in the /Services directory

	> `/Services/ProductApiDTO.cs `
	
	```csharp
	namespace APIDemo.MVCClient.Services;
	
	using System.Text.Json.Serialization;
	
	/* Swagger Schema "Product"
	{
	    id	integer($int32)
	    name	string
	    nullable: true
	    description	string
	    nullable: true
	    price	number($double)
	}
	
	Make sure the property decorators match the schema
	*/
	
	public class ProductApiDTO
	{
	    [JsonPropertyName("id")]
	    public int Id { get; set; }
	
	    [JsonPropertyName("name")]
	    public string? Name { get; set; }
	
	    [JsonPropertyName("description")]
	    public string? Description { get; set; }
	
	    [JsonPropertyName("price")]
	    public decimal Price { get; set; }
	}
	```

1.	Create a new file `ProductsApiService.cs` in the `Services` directory
	- Change the base URL to match the API

	> `/Services/ProductsApiService.cs `
	
	```csharp
	using System.Text.Json;
	
	namespace APIDemo.MVCClient.Services;
	
	public class ProductsApiService
	{
	    private readonly HttpClient _httpClient;
	    private readonly string _baseUrl = "http://localhost:5149/api/products";
	
	    public ProductsApiService(HttpClient httpClient)
	    {
	        _httpClient = httpClient;
	    }
	
	    public async Task<List<ProductApiDTO>> GetProductsAsync()
	    {
	        var response = await _httpClient.GetAsync(_baseUrl);
	        response.EnsureSuccessStatusCode();
	        var content = await response.Content.ReadAsStringAsync();
	        return JsonSerializer.Deserialize<List<ProductApiDTO>>(content);
	    }
	}
	```

1.	Add the `ProductsApiService` to the DI container in `Program.cs`

	> `/Program.cs `
	
	```csharp
	using APIDemo.MVCClient.Services;
	
	...

	builder.Services.AddHttpClient<ProductsApiService>();
	
	...


	```
	
### Controller

1. Go back to `ProductsController.cs` and add a constructor to get the HTTP Client and use the Service in the Index action metod

	> `/Controllers/ProductsController.cs `
	
	```csharp
	using APIDemo.MVCClient.Models;
	using APIDemo.MVCClient.Services;
	using Microsoft.AspNetCore.Mvc;
	
	namespace APIDemo.MVCClient.Controllers;
	
	public class ProductsController : Controller
	{
	    private readonly ProductsApiService _productsApiService;
	
	    public ProductsController(ProductsApiService productsApiService)
	    {
	        _productsApiService = productsApiService;
	    }
	
	    public async Task<IActionResult> Index()
	    {
	        // Get all products from the API service
	        var productsFromApiService = await _productsApiService.GetProductsAsync();
	
	        // Map the API DTO to the api model
	        var products = productsFromApiService.Select(p => new ProductViewModel
	        {
	            Id = p.Id,
	            Name = p.Name,
	            Description = p.Description,
	            Price = p.Price
	        });
	
	        // Mocked API data
	        //var products = GetProducts();
	
	        return View(products);
	    }
	
	    // Mocked API data. Private method to simulate API call
	    private List<ProductViewModel> GetProducts()
	    {
	        return new List<ProductViewModel>
	        {
	            new ProductViewModel
	            {
	                Id = 1,
	                Name = "Product 1",
	                Description = "Description for Product 1",
	                Price = 10.99m
	            },
	            new ProductViewModel
	            {
	                Id = 2,
	                Name = "Product 2",
	                Description = "Description for Product 2",
	                Price = 20.99m
	            },
	            new ProductViewModel
	            {
	                Id = 3,
	                Name = "Product 3",
	                Description = "Description for Product 3",
	                Price = 30.99m
	            }
	        };
	    }
	}
	```	

### Run the client app

1.	Make sure that the API application is started
1.	Run the client


## Use a static web app as an API Client

Create a subfoder `APIDemo.StaticWebClient`


### Static web app

1.	Create a new index file `index.html`

	> `/index.html`
	
	```html
	<!DOCTYPE html>
	<html lang="en">
	
	<head>
	    <meta charset="UTF-8">
	    <meta name="viewport" content="width=device-width, initial-scale=1.0">
	    <title>Products</title>
	    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.0/dist/css/bootstrap.min.css" rel="stylesheet">
	    <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.7.0/font/bootstrap-icons.css" rel="stylesheet">
	    <link rel="stylesheet" href="styles.css">
	</head>
	
	<body>
	    <div class="container">
	        <h1 class="text-center mt-5">Products</h1>
	        <button id="createProduct" class="btn btn-primary mt-3 mb-3">Create Product</button>
	        <table id="products" class="table table-bordered">
	            <thead>
	                <tr>
	                    <th>Id</th>
	                    <th>Name</th>
	                    <th>Description</th>
	                    <th>Price</th>
	                    <th>Actions</th>
	                </tr>
	            </thead>
	            <tbody>
	            </tbody>
	        </table>
	    </div>
	    <script src="app.js"></script>
	</body>
	
	</html>

	```

1.	Create a new app file `app.js`
	- Change the base URL to match the API `fetch('http://localhost:5083/api/Products')`

	> `/app.js `
	
	```javascript
	// Get the products table body
	var productsTbody = document.getElementById('products').getElementsByTagName('tbody')[0];
	
	// Fetch the products from the API
	fetch('http://localhost:5083/api/Products')
	    .then(response => response.json())
	    .then(products => {
	        // Iterate through the products and display them
	        products.forEach(product => {
	            var row = productsTbody.insertRow();
	            
	            var cell = row.insertCell(0);
	            cell.textContent = product.id;
	            
	            cell = row.insertCell(1);
	            cell.textContent = product.name;
	            
	            cell = row.insertCell(2);
	            cell.textContent = product.description;
	            
	            cell = row.insertCell(3);
	            cell.textContent = product.price;
	            
	            cell = row.insertCell(4);
	            
	            var editButton = document.createElement('button');
	            editButton.className = 'btn btn-secondary btn-sm me-2';
	            editButton.innerHTML = '<i class="bi bi-pencil"></i>';
	            editButton.addEventListener('click', function() {
	                editProduct(product.id);
	            });
	            
	            var deleteButton = document.createElement('button');
	            deleteButton.className = 'btn btn-danger btn-sm';
	            deleteButton.innerHTML = '<i class="bi bi-trash"></i>';
	            deleteButton.addEventListener('click', function() {
	                deleteProduct(product.id);
	            });
	            
	            cell.appendChild(editButton);
	            cell.appendChild(deleteButton);
	        });
	    })
	    .catch(error => console.error('Error:', error));
	
	// Get the create product button
	var createProductButton = document.getElementById('createProduct');
	
	// Add an event listener to the create product button
	createProductButton.addEventListener('click', function() {
	    createProduct();
	});
	
	function createProduct() {
	    // TODO: Implement the create product functionality
	    console.log('Create product');
	}
	
	function editProduct(id) {
	    // TODO: Implement the edit product functionality
	    console.log('Edit product', id);
	}
	
	function deleteProduct(id) {
	    // TODO: Implement the delete product functionality
	    console.log('Delete product', id);
	}

	```

1.	Create a new css file `styles.css`

	> `/styles.css `
	
	```css
	.product {
	    display: flex;
	    justify-content: space-between;
	    align-items: center;
	    padding: 10px;
	}

	```


### CORS

1.	Add the CORS Middleware to the pipeline in `Program.cs`

	> `/Program.cs `
	
	```csharp
	...
	
	// Set CORS options for policy
	builder.Services.AddCors(options =>
	{
	    options.AddPolicy("AllowAllOrigins", builder =>
	    {
	        builder.AllowAnyOrigin()
	                .AllowAnyHeader()
	                .AllowAnyMethod();
	    });
	});

	...

	// Use CORS policy
	app.UseCors("AllowAllOrigins");

	...
	```

### Run the client app

1.	Make sure that the API application is started
1.	Run the client by opening the index.html file in a browser

