+++
title = 'Deploy a Webapp on Google CloudRun'
weight = 5
date = 2024-09-02
draft = false
+++

## Demo: Auth Authz - Cloud Run

https://cloud.google.com/run/docs/overview/what-is-cloud-run

### Prepare and make sure all services work at localhost

1. Open up service providers

	- GCP Credentials
	- GCP CloudRun
	- AWS Cognito

1. Make sure all credentials are correct

	- appsettings.json (MetadataAddress, CognitoDomain)
	- user-secrets
	
		```bash
		dotnet user-secrets init
		dotnet user-secrets list
		dotnet user-secrets set "Authentication:Cognito:ClientId" "COGNITO_CLIENT_ID_GOES_HERE"
		dotnet user-secrets set "Authentication:Google:ClientId" "GOOGLE_CLIENT_ID_GOES_HERE"
		dotnet user-secrets set "Authentication:Google:ClientSecret" "GOOGLE_CLIENT_SECRET_GOES_HERE"
		```
1. Run the app locally with https:
	- `dotnet run --launch-profile https`
	- `dotnet run --urls "https://localhost:5001"`

1. Check-in and push the repo to Github


### Adjust the code to work behind a reverse proxy

1. Configure ForwardedHeaders middleware in `Program.cs`
	
	> `/Program.cs`
	
	```csharp
	...
	using Microsoft.AspNetCore.HttpOverrides;
	
	...
	builder.Services.Configure<ForwardedHeadersOptions>(options =>
	{
	    options.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
	    options.KnownNetworks.Clear();
	    options.KnownProxies.Clear();
	});
	
	...
	app.UseForwardedHeaders();
	
	...
	```

### Adjust the code to work on Google CloudRun

> Optional: GCP sets an environment variable PORT. An alternative is to set the ASPNETCORE_URLS to the same port.

1. Make it possible to set the port through an environment variable in `Program.cs`. Google Cloud Run uses this to route traffic to the application running inside a container in the service
	
	> `/Program.cs`
	
	```csharp
	...
	// Use the PORT environment variable to configure the application to listen on a specific port.
	var port = Environment.GetEnvironmentVariable("PORT") ?? "";
	if (!string.IsNullOrEmpty(port))
	{
	    app.Urls.Add($"http://*:{port}");
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
	ASPNETCORE_URLS=http://*:5000
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

