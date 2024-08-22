+++
title = 'Google and Cognito Authentication'
weight = 3
date = 2024-08-22
draft = false
+++

## Demo: Auth Authz - Google

https://learn.microsoft.com/en-us/aspnet/core/security/authentication/social/google-logins?view=aspnetcore-7.0

### Prepare Development Certificate

In this tutorial you need to use the development certificate in order to start your application on https.



**Different options to start in https**

1) Use launchSettings.json profiles

```bash
dotnet run --launch-profile "https"
```

2) Use an urgument to dotnet run

```bash
dotnet run --urls "https://localhost:5001"
```

3) Configure Kestrel in Program.cs or appsettings.json

> Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.WebHost.UseUrls("https://localhost:5001");

var app = builder.Build();

// Configure the HTTP request pipeline.

app.Run();
```

> appsettings.json

```json
Kopiera kod
{
  "Kestrel": {
    "Endpoints": {
      "Https": {
        "Url": "https://localhost:5001"
      }
    }
  }
}
```


### Setup Google

Prepare the Google Credentials API

1. Fill in the OAuth Consent information
	- Select "External"

1. Create Credentials
	- OAuth client ID
	- Web Application
	- Authorized redirect URIs: 
		- `https://localhost:7131/signin-google`

```bash
dotnet user-secrets init
dotnet user-secrets set "Authentication:Google:ClientId" "GOOGLE_CLIENT_ID_GOES_HERE"
dotnet user-secrets set "Authentication:Google:ClientSecret" "GOOGLE_CLIENT_SECRET_GOES_HERE"
```


### Configure Google authentication

```bash
dotnet add package Microsoft.AspNetCore.Authentication.Google
```


1. Configure Authentication middleware in `Program.cs`
	
	> `/Program.cs`
	
	```csharp
	...
	
	builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
	    .AddCookie()
	    .AddGoogle(options =>
	    {
	        options.ClientId = builder.Configuration["Authentication:Google:ClientId"] ?? throw new ArgumentNullException("Authentication:Google:ClientId");
	        options.ClientSecret = builder.Configuration["Authentication:Google:ClientSecret"] ?? throw new ArgumentNullException("Authentication:Google:ClientSecret");
	    });

	...
	
	
	```

### Controller

1. Add two new actions `GoogleLogin()` and `GoogleLoginCallbackAsync()` in `AccountController.cs`

	> `/Controllers/AccountController.cs`
	
	```csharp
	...
	using Microsoft.AspNetCore.Authentication.Google;
	...
	
    public IActionResult GoogleLogin()
    {
        var authProperties = new AuthenticationProperties 
        {
            RedirectUri = Url.Action("GoogleLoginCallback", "Account")
        };
        return Challenge(authProperties, GoogleDefaults.AuthenticationScheme);
    }

    public async Task<IActionResult> GoogleLoginCallbackAsync()
    {
        var result = await HttpContext.AuthenticateAsync(CookieAuthenticationDefaults.AuthenticationScheme);
        if (!result.Succeeded)
        {
            // Handle failure: return to the login page, show an error, etc.
            return RedirectToAction("Login");
        }

        // Here, you could fetch information from result.Principal to store in your database, 
        // or to find an existing user.

        return RedirectToAction("Index", "Home");
    }

	```	
	
### View

1. Add a new button `Login with Google` in the view file `Login.cshtml`
	
	> `/Views/Account/Login.cshtml`
	
	```html
	
	...
	
	<a href="@Url.Action("GoogleLogin", "Account")" class="btn btn-primary">Login with Google</a>

	
	...
	
	```
	
	
# Demo: Auth Authz - Cognito

## Setup Cognito

1. Go to Cognito
2. Create User Pool
3. Select User name -> Next
4. Select No MFA -> Next
5. Leave default -> Next
6. Select Send email with Cognito -> Next
7. Enter User pool name
8. Select Use the Cognito Hosted UI
9. Enter a Cognito domain (like same as userpool)
10. Enter App client name
11. Enter Allowed callback URLs (https://localhost:7131/signin-oidc) (Use your port!)
12. Expand Advanced app client setttings
13. Add _Profile_ to _OpenID Connect scopes_
14. Add Allowed sign-out URLs (https://localhost:7131/) -> Next
15. Review -> Create user pool



### Configure Cognito authentication

```bash
dotnet add package Microsoft.AspNetCore.Authentication.OpenIdConnect
```


1. Configure Authentication middleware in `Program.cs`
	
	> `/Program.cs`
	
	```csharp
	...
	
	using Microsoft.AspNetCore.Authentication.OpenIdConnect;
	
	...
	
	builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
	    .AddCookie()
	    .AddGoogle(options =>
	    {
	        options.ClientId = builder.Configuration["Authentication:Google:ClientId"] ?? throw new InvalidOperationException("Google ClientId is not set.");
	        options.ClientSecret = builder.Configuration["Authentication:Google:ClientSecret"] ?? throw new InvalidOperationException("Google ClientSecret is not set.");
	    })
	    .AddOpenIdConnect(options =>
	    {
	        options.ClientId = builder.Configuration["Authentication:Cognito:ClientId"] ?? throw new InvalidOperationException("Cognito ClientId is not set.");
	        options.ResponseType = builder.Configuration["Authentication:Cognito:ResponseType"] ?? throw new InvalidOperationException("Cognito ResponseType is not set.");
	        options.MetadataAddress = builder.Configuration["Authentication:Cognito:MetadataAddress"] ?? throw new InvalidOperationException("Cognito MetadataAddress is not set.");
	        options.Events = new OpenIdConnectEvents
	        {
	            OnRedirectToIdentityProviderForSignOut = context =>
	            {
	                context.ProtocolMessage.Scope = builder.Configuration["Authentication:Cognito:Scope"] ?? throw new InvalidOperationException("Cognito Scope is not set.");
	                context.ProtocolMessage.ResponseType = builder.Configuration["Authentication:Cognito:ResponseType"] ?? throw new InvalidOperationException("Cognito ResponseType is not set.");;
	                // context.ProtocolMessage.IssuerAddress = CognitoHelpers.GetCognitoLogoutUrl(builder.Configuration, context.HttpContext);
	
	                // Create Cognito logout URL
	                var cognitoDomain = builder.Configuration["Authentication:Cognito:CognitoDomain"] ?? throw new InvalidOperationException("Cognito CognitoDomain is not set.");
	                var clientId = builder.Configuration["Authentication:Cognito:ClientId"] ?? throw new InvalidOperationException("Cognito ClientId is not set.");
	                var appSignOutUrl = builder.Configuration["Authentication:Cognito:AppSignOutUrl"] ?? throw new InvalidOperationException("Cognito AppSignOutUrl is not set.");
	                var logoutUrl = $"{context.Request.Scheme}://{context.Request.Host}{appSignOutUrl}";
	                var cognitoLogoutUrl = $"{cognitoDomain}/logout?client_id={clientId}&logout_uri={logoutUrl}";
	
	                context.ProtocolMessage.IssuerAddress = cognitoLogoutUrl;
	
	                // Close authentication sessions
	                context.Properties.Items.Remove(CookieAuthenticationDefaults.AuthenticationScheme);
	                context.Properties.Items.Remove(OpenIdConnectDefaults.AuthenticationScheme);
	
	                return Task.CompletedTask;
	            }
	        };
	    });


	...
	
	
	```
	


### Appsettings

Go to the _App integration_ tab

1. Modify MetadataAddress with _AWS Region_ and _UserPoolID_ (User pool overview)
2. Set the Cognito Domain
3. Set the ClientID
	- `dotnet user-secrets set "Authentication:Cognito:ClientId" "COGNITO_CLIENT_ID_GOES_HERE"`

> `/appsettings.json`
	
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "Authentication": {
    "Cognito": {
      "ClientId": "COGNITO_CLIENT_ID_GOES_HERE",
      "ResponseType": "code",
      "MetadataAddress": "https://cognito-idp.eu-west-1.amazonaws.com/eu-west-1_dKrElOUfh/.well-known/openid-configuration",
      "Scope": "openid",
      "AppSignOutUrl": "/",
      "CognitoDomain": "https://aspnetintegration.auth.eu-west-1.amazoncognito.com"
    },
    "Google": {
      "ClientId": "GOOGLE_CLIENT_ID_GOES_HERE",
      "ClientSecret": "GOOGLE_CLIENT_SECRET_GOES_HERE"
    }
  }
}
```

### Controller

1. Add a new action `CognitoLogin()` in `AccountController.cs`

	> `/Controllers/AccountController.cs`
	
	```csharp
	...
	using Microsoft.AspNetCore.Authentication.OpenIdConnect;
	...
	
    public IActionResult CognitoLogin()
    {
        // Challenge the Cognito authentication scheme
        return Challenge(
            new AuthenticationProperties
            {
                RedirectUri = Url.Action("Index", "Home")
            },
            OpenIdConnectDefaults.AuthenticationScheme);
    }

	```	

2. Add `OpenIdConnectDefaults.AuthenticationScheme` to Signout

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
            CookieAuthenticationDefaults.AuthenticationScheme,
            OpenIdConnectDefaults.AuthenticationScheme);
    }

	```	

	
### View

1. Add a new button `Login with Cognito` in the view file `Login.cshtml`
	
	> `/Views/Account/Login.cshtml`
	
	```html
	
	...
	
	<a href="@Url.Action("CognitoLogin", "Account")" class="btn btn-primary" >Login with Cognito</a>

	
	...
	
	```
	
