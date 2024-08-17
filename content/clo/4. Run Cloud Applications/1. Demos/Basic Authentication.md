+++
title = 'Basic Authentication'
weight = 2
date = 2024-08-17
draft = false
+++

## Demo: Auth Authz

Start a new MVC project

```bash
dotnet new mvc
```

### Model

1.	Create a new model file `LoginViewModel.cs` in the /Models directory

	> `/Models/LoginViewModel.cs `
	
	```csharp
	using System.ComponentModel.DataAnnotations;
	
	namespace AuthDemo.Models;
	
	public class LoginViewModel
	{
	    [Required]
	    public string? Username { get; set; }
	
	    [Required]
	    [DataType(DataType.Password)]
	    public string? Password { get; set; }
	}
	```

### Controller

1. Create a new controller `AccountController.cs` and add a new action `Login()`

	> `/Controllers/AccountController.cs`
	
	```csharp
	using Microsoft.AspNetCore.Mvc;
	using AuthDemo.Models;
	
	namespace AuthDemo.Controllers;
	
	public class AccountController : Controller
	{
	    // Mocked user data
	    private const string MockedUsername = "demo";
	    private const string MockedPassword = "pass"; // Note: NEVER hard-code passwords in real applications.
	
	    public IActionResult Login()
	    {
	        return View();
	    }
	
	    [HttpPost]
	    [ValidateAntiForgeryToken] // This ensures that the form is submitted with a valid anti-forgery token to prevent CSRF attacks.
	    public IActionResult Login(LoginViewModel model)
	    {
	        // Check model validators
	        if (!ModelState.IsValid)
	        {
	            return View(model);
	        }
	
	        // Mocked user verification
	        if (model.Username == MockedUsername && model.Password == MockedPassword)
	        {
	            // Normally, here you'd set up the session/cookie for the authenticated user.
	            return RedirectToAction("Index", "Home"); // Redirect to a secure area of your application.
	        }
	
	        ModelState.AddModelError(string.Empty, "Invalid login attempt."); // Generic error message for security reasons.
	        return View(model);
	    }
	}
	```	
	
### View

1. Create a new directory `Account` and a new view file `Login.cshtml`
	
	> `/Views/Account/Login.cshtml`
	
	```html
	@model AuthDemo.Models.LoginViewModel
	
	<h2>Login</h2>
	
	<form asp-action="Login" asp-controller="Account" method="post" asp-antiforgery="true">
	    <div asp-validation-summary="ModelOnly" class="text-danger"></div>
	
	    <div>
	        <label asp-for="Username">Username:</label>
	        <input type="text" id="Username" asp-for="Username" required />
	        <span asp-validation-for="Username" class="text-danger"></span>
	    </div>
	    
	    <div>
	        <label asp-for="Password">Password:</label>
	        <input type="password" id="Password" asp-for="Password" required />
	        <span asp-validation-for="Password" class="text-danger"></span>
	    </div>
	
	    <div>
	        <button type="submit">Login</button>
	    </div>
	</form>
	
	@section Scripts {
	    @{await Html.RenderPartialAsync("_ValidationScriptsPartial");}
	}
	```
	

### Navigation

1.	Add a button to the menu bar in the file `_Layout.cshtml` in the `/Views/Shared` directory

	> `/Views/Shared/_Layout.cshtml `
	
	```html
	<li class="nav-item">
	    <a class="nav-link text-dark" asp-area="" asp-controller="Account" asp-action="Login">Login</a>
	</li>
	```
	
	
## Add Authentication

### Controller

1. Add a new action `SecretInfo()` to the AccountController

	> `/Controllers/AccountController.cs`
	
	```csharp
	...
	
	using Microsoft.AspNetCore.Authorization;
	
	...
	
    public IActionResult SecretInfo()
    {
        return View();
    }
   
	```	

### View

1. Create a new view file `SecretInfo.cshtml`
	
	> `/Views/Account/SecretInfo.cshtml`
	
	```html
	<h2>Authentication Info</h2>
	
	<table class="table">
	    <thead>
	        <tr>
	            <th>Claim Type</th>
	            <th>Claim Value</th>
	        </tr>
	    </thead>
	    <tbody>
	        @foreach (var claim in User.Claims)
	        {
	            <tr>
	                <td>@claim.Type</td>
	                <td>@claim.Value</td>
	            </tr>
	        }
	    </tbody>
	</table>
	```

### Navigation

1.	Add a button to the menu bar in the file `_Layout.cshtml` in the `/Views/Shared` directory

	> `/Views/Shared/_Layout.cshtml `
	
	```html
    <li class="nav-item">
        <a class="nav-link text-dark" asp-area="" asp-controller="Account" asp-action="SecretInfo">Secret Info</a>
    </li>
	```

### Authenticate and login the user

1. Add and configure Authentication middleware in `Program.cs`
	
	> `/Program.cs`
	
	```csharp
	using Microsoft.AspNetCore.Authentication.Cookies;
	
	...
	
	builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
   		.AddCookie();

	...
	
	app.UseAuthentication(); // Before UseAuthorization()
	
	...
	
	```

1. Protect the SecretInfo page in the `AccountController.cs`

	The login page will be found based on naming convention `/Account/Login`
	
	> `/Controllers/AccountController.cs`
	
	```csharp
	...
	
	using Microsoft.AspNetCore.Authorization;
	
	...
	
	[Authorize] // This attribute ensures that only authenticated users can access this action.
	public IActionResult SecretInfo()
	{
	    return View();
	}
	
	...	
	```

1. Set up the session/cookie for the authenticated user in the `AccountController.cs`
	
	> `/Controllers/AccountController.cs`
	
	```csharp
	...
	
	using System.Security.Claims;
	using Microsoft.AspNetCore.Authentication.Cookies;
	using Microsoft.AspNetCore.Authentication;
	
	...
	
    [HttpPost]
    [ValidateAntiForgeryToken] // This ensures that the form is submitted with a valid anti-forgery token to prevent CSRF attacks.
    public async Task<IActionResult> LoginAsync(LoginViewModel model)
    {
        // Check model validators
        if (!ModelState.IsValid)
        {
            return View(model);
        }

        // Mocked user verification
        if (model.Username == MockedUsername && model.Password == MockedPassword)
        {
            // Set up the session/cookie for the authenticated user.
            var claims = new[] { new Claim(ClaimTypes.Name, model.Username) };
            var identity = new ClaimsIdentity(claims, CookieAuthenticationDefaults.AuthenticationScheme);
            var principal = new ClaimsPrincipal(identity);

            await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, principal);
            // Normally, here you'd set up the session/cookie for the authenticated user.
            return RedirectToAction("Index", "Home"); // Redirect to a secure area of your application.
        }

        ModelState.AddModelError(string.Empty, "Invalid login attempt."); // Generic error message for security reasons.
        return View(model);
    }
	
	...	
	```

### Log out the user

	
1.	Change the login button in the menu bar in the file `_Layout.cshtml` in the `/Views/Shared` directory so that it toggles between `Login` and `Logout`

	> `/Views/Shared/_Layout.cshtml `
	
	```html
	...
	
	@if (Context.User.Identity.IsAuthenticated)
	{
	    <a class="nav-link text-dark" asp-area="" asp-controller="Account" asp-action="Logout">Logout</a>
	}
	else
	{
	    <a class="nav-link text-dark" asp-area="" asp-controller="Account" asp-action="Login">Login</a>
	}
	
	...
	
	```

1. Add a new action `Logout()` to the AccountController

	
	> `/Controllers/AccountController.cs`
	
	```csharp
	...
	
    [Authorize]
    public IActionResult Logout()
    {
        return SignOut(
            new AuthenticationProperties
            {
                RedirectUri = Url.Action("Index", "Home")
            },
            CookieAuthenticationDefaults.AuthenticationScheme);
    }

	```

