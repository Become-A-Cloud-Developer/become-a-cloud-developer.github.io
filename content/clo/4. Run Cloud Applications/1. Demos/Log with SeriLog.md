+++
title = 'Log with SeriLog'
weight = 6
date = 2024-09-02
draft = false
+++

## Demo: Auth Authz - Logging

### Prepare and make sure all services work at localhost

1. Run the app locally with https:
	- `dotnet run --launch-profile https`
	- `dotnet run --urls "https://localhost:5001"`


### Add SeriLog

With this implementation we use structured logging with json and add a new property "UserName" to each log entry indicating who is logged in.

1. Install Nuget packages:

	```bash
	dotnet add package Serilog.AspNetCore
	dotnet add package Serilog.Enrichers.AspNetCore
	```

1. Add the OnTokenValidated eventhandler to the AddOpenIdConnect authentication midleware in `Program.cs`. We need this to map one of the claims of the logged in user from cognito to a "Name" claim, which Cognito does not have by default.
	
	> `/Program.cs`
	
	```csharp
	...
	using System.Security.Claims;
	
	using Serilog;
	using Serilog.Core;
	using Serilog.Formatting.Compact;
	using Utilities.Logging;
	
	...
	builder.Services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
	    .AddCookie()
	    .AddGoogle(options =>
	    {
	        options.ClientId = builder.Configuration["Authentication:Google:ClientId"] ?? throw new ArgumentNullException("Authentication:Google:ClientId");
	        options.ClientSecret = builder.Configuration["Authentication:Google:ClientSecret"] ?? throw new ArgumentNullException("Authentication:Google:ClientSecret");
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
	            },
	            OnTokenValidated = context =>
	            {
	                var claims = context.Principal.Claims
	                    .Append(new Claim(ClaimTypes.Name, context.Principal.FindFirst("cognito:username").Value));
	
	                var claimsIdentity = new ClaimsIdentity(claims, context.Scheme.Name, ClaimsIdentity.DefaultNameClaimType, ClaimsIdentity.DefaultRoleClaimType);
	
	                context.Principal = new ClaimsPrincipal(claimsIdentity);
	
	                return Task.CompletedTask;
	            }
	        };
	    });
	
	...
	builder.Services.AddHttpContextAccessor();
	builder.Services.AddSingleton<UserNameEnricher>();
	
	var serviceProvider = builder.Services.BuildServiceProvider();
	var userNameEnricher = serviceProvider.GetService<UserNameEnricher>();
	
	// Configure Serilog
	Log.Logger = new LoggerConfiguration()
	    .Enrich.FromLogContext() // Enrich log messages with additional context (e.g., request information).
	    .Enrich.With(userNameEnricher) // Enrich log messages with the current user name.
	    .WriteTo.Console(new RenderedCompactJsonFormatter()) // Output logs in JSON format.
	    .CreateLogger();
	
	// Override the default logger configuration with Serilog.
	builder.Logging.ClearProviders();
	builder.Logging.AddSerilog(Log.Logger);

	...
	app.Run();
	
	...
	// Flush and close the log.
	Log.CloseAndFlush();
	```

1. Add a new directory `/Utilities/Logging` and a new file `UserNameEnricher.cs`. This is a class that reads the name of the logged in identity from the HTTPContext and add (enrich) that information to every log entry
	
	> `/UserNameEnricher.cs`
	
	```csharp
	using Serilog.Core;
	using Serilog.Events;
	
	namespace Utilities.Logging;
	
	public class UserNameEnricher : ILogEventEnricher
	{
	    private readonly IHttpContextAccessor _httpContextAccessor;
	
	    public UserNameEnricher(IHttpContextAccessor httpContextAccessor)
	    {
	        _httpContextAccessor = httpContextAccessor;
	    }
	
	    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
	    {
	        var userName = _httpContextAccessor.HttpContext?.User?.Identity?.Name;
	        var userNameProperty = new LogEventProperty("UserName", new ScalarValue(userName));
	        logEvent.AddPropertyIfAbsent(userNameProperty);
	    }
	}

	```

1. Let´s add a logging statement in the AccountController when accessing the SecretInfo method.
	- Get the logger from dependency injection in the constructor

	> `/Controllers/AccountController.cs`
	
	```csharp
	...
	public class AccountController : Controller
	{
	    private readonly ILogger<AccountController> _logger;
	
	    // Mocked user data
	    private const string MockedUsername = "demo";
	    private const string MockedPassword = "pass"; // Note: NEVER hard-code passwords in real applications.
	
	    public AccountController(ILogger<AccountController> logger)
	    {
	        _logger = logger;
	    }

	...
	    public IActionResult SecretInfo()
	    {
	        _logger.LogInformation("Secret info page requested.");
	
	        return View();
	    }
	
	...
	

	```


## Setup Google Cloud Run

1. Go to Cloud Run
2. Create Service
3. Select Continuously deploy new revisions from a source repository
4. Press the button: SET UP WITH CLOUD BUILD
5. Select Github as Repository Provider
6. Select Repository -> Next
	- Here you might need to install a plugin on Github - follow the instructions in Manage connected repositories
7. Select the Branch
8. Select Build Type Google Cloud's buildpacks -> Save
	- Go, Node.js, Python, Java, .NET Core, Ruby or PHP via Google Cloud's buildpacks
9. Enter a Service name (same as git repo or similar)
10. Select Region (Finland)
11. Select Allow unauthenticated invocations
12. Expand Container, Networking, Security
13. Press the button + ADD VARIABLE and enter the secrets

	```bash
	Authentication__Google__ClientSecret
	Authentication__Google__ClientId
	Authentication__Cognito__ClientId
	```

14. Go to the NETWORKING tab
15. Select Session affinity
16. Press CREATE

Note the URL for the Cloud Run app: `https://authdemo2-7v7mzttyba-lz.a.run.app`

You need this later

Verify that the application runs properly by pasting the URL into a browser. (The Google and Cognito Login will not work yet)


### Update the Google Credentials configuration


1. Go to the Google Credentials service
2. Press + ADD URI under Authorized redirect URIs
	- `https://authdemo2-7v7mzttyba-lz.a.run.app/signin-google`
1. Save

1. Open a new private browser window
	- Try the Login with Google on the login page (logout will not work properly yet)


### Update the AWS Cognito configuration


1. Go to the AWS Cognito service
2. Choose User pools and select your pool
3. Go to the tab App integration and then all the way to the bottom select your App client
4. Press Edit in the Hosted UI section
5. Press the button Add another URL under Allowed callback URLs
	- `https://authdemo2-7v7mzttyba-lz.a.run.app/signin-oidc`
5. Press the button Add another URL under Allowed sign-out URLs
	- `https://authdemo2-7v7mzttyba-lz.a.run.app/`
1. Press Save changes

1. Open a new private browser window
	- Try the Login with Google and Login with Cognito on the login page

