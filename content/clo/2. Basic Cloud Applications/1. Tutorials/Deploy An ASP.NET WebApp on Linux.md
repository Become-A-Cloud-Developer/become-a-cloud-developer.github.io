+++
title = "How to Deploy An ASP.NET WebApp on Linux"
weight = 2
date = 2024-02-14
draft = false
+++

# How to Deploy An ASP.NET WebApp on Linux

## Introduction

This tutorial will guide you through deploying an ASP.NET Core web application on an Ubuntu 22.04 LTS Linux VM, running on Azure.

The tutorial is based on three articles from Microsoft Learn

**Register the Microsoft package repository:**

https://learn.microsoft.com/en-us/dotnet/core/install/linux-ubuntu#register-the-microsoft-package-repository

**Install the runtime on Ubuntu 22.04:**

https://learn.microsoft.com/en-us/dotnet/core/install/linux-ubuntu-2204#install-the-runtime

**Create a service file:**

https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/linux-nginx?view=aspnetcore-8.0&tabs=linux-ubuntu#create-the-service-file

## Method

- Provision a VM running Ubuntu 22.04 LTS as the Operating System (OS) on Azure.
- Install the .NET runtime (not the SDK) from the Microsoft Package Repository
- Create a SystemD service file (unit file) in order to run the web app as a service
- We will access the server using a locally installed terminal (GitBash on Windows, Terminal on Mac and Linux) and use SCP to copy the web app artifacts to the server.

## Prerequisites

- An Azure account. If you don't have one, sign up [here](https://azure.microsoft.com/).
- A locally developped ASP.NET web application, ready for deployment
- For Windows users: GitBash or a similar terminal. Mac and Linux users can use the pre-installed terminal.

## Provision a Virtual Machine

1. Log into Azure Portal
2. Create the VM (Ubuntu 22.04 LTS)

> Note: Select a region close to you.
> 
> The "Standard_B1s" size is suitable for this tutorial.
> 
> Open the firewall on port 5000 (this is where ASP.NET web apps listen by default)

## Install .NET on Ubuntu

1. **Register the Microsoft Package Repository**

	```bash
	# Get Ubuntu version
	declare repo_version=$(if command -v lsb_release &> /dev/null; then lsb_release -r -s; else grep -oP '(?<=^VERSION_ID=).+' /etc/os-release | tr -d '"'; fi)
	
	# Download Microsoft signing key and repository
	wget https://packages.microsoft.com/config/ubuntu/$repo_version/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
	
	# Install Microsoft signing key and repository
	sudo dpkg -i packages-microsoft-prod.deb
	
	# Clean up
	rm packages-microsoft-prod.deb
	
	# Update packages
	sudo apt update
	```

	Reference: instructions from [Microsoft's documentation](https://learn.microsoft.com/en-us/dotnet/core/install/linux-ubuntu#register-the-microsoft-package-repository) to add Microsoft's package repository.

2. **Install the .NET Runtime and SDK**

	```bash
	sudo apt-get update && sudo apt-get install -y aspnetcore-runtime-8.0
	```

   Reference: The guide provided by Microsoft for [installing the runtime on Ubuntu 22.04](https://learn.microsoft.com/en-us/dotnet/core/install/linux-ubuntu-2204#install-the-runtime).

3. **Verify installation**

	```bash
	dotnet --list-runtimes
	```

## Deploy the ASP.NET Core Application

1. **Publish Your ASP.NET Core App**
   - Use the `dotnet publish` command to package your local application.

2. **Transfer the Application to Your VM**
   - Use `scp` or a similar tool to copy the published application to your VM.
   - It's very common to deploy apps in the `/opt` directory.

**Example:**

1. Prepare a directory, for the web app files, on the server
	- Login to the server

	```bash
	sudo mkdir -p /opt/myapp
	sudo chown azureuser:azureuser /opt/myapp
	```

2. Copy the app artifact files to the server
	- Open en terminal in `publish` directory where you have the web app artifacts

	```bash
	scp -r -i <private_key> ./ azureuser@<external_ip>:/opt/myapp
	```

## Create a Service File for Your ASP.NET Core Application

To ensure your ASP.NET Core application runs as a service on Ubuntu, you'll need to create a systemd service file. This service will start your application at boot and keep it running in case of failure. Here’s how you can set it up:

1. **Create a Service File**
   
   Open a terminal and use your preferred text editor to create a new service file in the `/etc/systemd/system` directory. The file should be named after your application, for example, `myapp.service`.

   ```bash
   sudo nano /etc/systemd/system/myapp.service
   ```

2. **Define the Service File**

   Paste the following content into the editor, adjusting the paths and settings to match your application's requirements. Here’s an example service file for an ASP.NET Core application:

   ```ini
   [Unit]
   Description=ASP.NET Web App running on Ubuntu

   [Service]
   WorkingDirectory=/opt/myapp
   ExecStart=/usr/bin/dotnet /opt/myapp/myapp.dll
   Restart=always
   RestartSec=10
   KillSignal=SIGINT
   SyslogIdentifier=myapp
   User=www-data
   Environment=ASPNETCORE_ENVIRONMENT=Production
   Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false
   Environment="ASPNETCORE_URLS=http://*:5000"

   [Install]
   WantedBy=multi-user.target
   ```
   
   Reference: The guide provided by Microsoft for [Creating a service file](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/linux-nginx?view=aspnetcore-8.0&tabs=linux-ubuntu#create-the-service-file).
   
   
   	> Note: 
	> 	
	> `Environment="ASPNETCORE_URLS=http://*:5000"`: Use environment variable to make the web app listen to the source and port you want
	>

3. **Enable and Start the Service**

   After saving the file, enable the service to start at boot and then start the service immediately:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable myapp.service
   sudo systemctl start myapp.service
   ```

4. **Verify the Service is Running**

   To check the status of your service, use:

   ```bash
   sudo systemctl status myapp.service
   ```

   This command provides information about the service’s state, including whether it’s active and running.

Creating a systemd service for your ASP.NET Core application ensures it's always running and starts automatically after a reboot or if the application crashes. This setup is crucial for maintaining the availability and reliability of your web application.

## Accessing Your Application

- Your ASP.NET Core application should be accessible via the public IP address of your VM on the port configured in your setup (`<IP>:<port>`).

## Troubleshooting

- **Firewall Issues**: Make sure Azure's network security group (NSG) rules allow traffic on the port your app uses.
- **Service Configuration**: Ensure your ASP.NET Core application's service is running with `systemctl status myapp.service`.
- Logs for troubleshooting:
  - `sudo journalctl -u myapp.service`

## Final Thoughts

There are several ways to install the .Net Runtime, but this method seems reliable and straight forward IMHO.

## Don't Forget

Azure services incur costs. Delete resources you no longer need.

# Happy Deploying! 🚀







