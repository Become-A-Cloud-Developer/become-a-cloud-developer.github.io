+++
title = "Containerize an ASP.NET Webapp"
weight = 9
date = 2024-03-22
draft = false
+++

## Introduction

In this step-by-step tutorial we will containerize an ASP.NET web app and deploy it on Azure. Containerization is a cornerstone in modern software development, offering flexibility and efficiency in deploying and managing applications. 

Throughout this guide, we'll use Docker, Azure, and various CLI tools to bring your ASP.NET application to the cloud. By the end of this tutorial, you'll have a solid understanding of containerization and be able to deploy your containerized applications on Azure.

## Prerequisites

In order to follow along, ensure you have the following:

- An active Azure account. If you don't have one, sign up [here](https://azure.microsoft.com/en-us/free/).
- Docker (Docker Desktop) installed on your local machine. [Download Docker](https://www.docker.com/products/docker-desktop).
- Visual Studio Code (VSCode) as your IDE. [Download VSCode](https://code.visualstudio.com/Download).
- .NET Command-Line Interface (CLI) for interacting with your project. [Learn more](https://docs.microsoft.com/en-us/dotnet/core/tools/).
- .NET SDK 8.0 installed to build .NET applications. [Install .NET](https://dotnet.microsoft.com/download).
- A Docker Hub account for storing your images. Sign up [here](https://hub.docker.com/).
- Azure CLI installed for managing Azure resources. [Install Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).

## Method

Deploying your app in a container involves three key steps:

1. **Build Image and Push It to a Registry**: Package your app into a Docker image and upload it to Docker Hub.
2. **Provision a Docker Host**: Prepare an Azure environment where your container will live.
3. **Deploy the Image and Run the Container**: Run your app in the cloud, making it accessible worldwide.

## Before We Begin

Let's set the stage with a simple ASP.NET Core web project named `DockerDemoApp`. This example will help you understand the process without the complexity of a real-world application. 

1. Create a directory for your project:

    ```bash
    mkdir DockerDemoApp
    cd DockerDemoApp
    ```

2. Initialize a new ASP.NET Core web application:

    ```bash
    dotnet new webapp
    ```

## Step 1: Containerize the App

Containerizing your app is the first step to deploying it on Azure. Below, we present three alternatives based on your preference and project needs.

### Alternative 1: Classic (Dockerfile)

This section is based on this [Microsoft Learn article](https://learn.microsoft.com/en-us/dotnet/core/docker/build-container?tabs=linux&pivots=dotnet-8-0)

This method involves manually creating a `Dockerfile` to describe your Docker image.

1. **Create a Dockerfile**: At your project's root (where the .csproj file is), create a `Dockerfile` with the following content:

	> Dockerfile

	```Dockerfile
	FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build-env
	WORKDIR /App

	# Copy everything
	COPY . ./
	# Restore as distinct layers
	RUN dotnet restore
	# Build and publish a release
	RUN dotnet publish -c Release -o out

	# Build runtime image
	FROM mcr.microsoft.com/dotnet/aspnet:8.0
	WORKDIR /App
	COPY --from=build-env /App/out .
	ENTRYPOINT ["dotnet", "DockerDemoApp.dll"]
	```

    This `Dockerfile` uses a multi-stage build process to create a lightweight image of your app.

2. **Build and Push Your Image**:

    - Build your Docker image:

        ```bash
        docker build -t <repo>/dockerdemoapp .
        ```
    
    - Push it to Docker Hub:

        ```bash
        docker login
        docker push <repo>/dockerdemoapp
        ```

3. **Verify Locally**: Run your image locally to ensure it works:

    ```bash
    docker run -d -p 8080:8080 --name MyDockerDemoApp <repo>/dockerdemoapp
    ```

    Open your browser and navigate to `http://localhost:8080` to see your app running.
    
    > Note: In .Net 8.0 there is a breaking change from previous versions. By default the webapp now listens to port 8080 inside a container, while in older .Net versions it listens to port 80.


### Alternative 2: Docker CLI

This section is based on this [Docker article](https://docs.docker.com/language/dotnet/containerize/)

For those who prefer less manual configuration, the Docker CLI can generate a Dockerfile for you.

1. **Initialize Docker**: In your project directory, run:

    ```bash
    docker init
    ```

    Follow the prompts to auto-generate your `Dockerfile`.

2. **Build and Push Your Image**: Similar to the classic method, build and push your image to Docker Hub.

### Alternative 3: Dotnet CLI (without Dockerfile)

This section is based on this [Github article](https://github.com/dotnet/dotnet-docker/blob/main/samples/aspnetapp/README.md)

This method uses the Dotnet CLI to build and publish your app directly to Docker Hub, skipping the Dockerfile.

1. **Publish Your App**:

    ```bash
    dotnet publish /p:PublishProfile=DefaultContainer /p:ContainerBaseImage=mcr.microsoft.com/dotnet/aspnet:8.0-jammy-chiseled /p:ContainerRegistry=docker.io /p:ContainerRepository=<repo>/dockerdemoapp
    ```

	> Note: If you use GitBash as your terminal you might need to change the syntax of the command to the one below. The difference is to use `-p ` instead of `/p:`
	> 
	> ```bash
	> dotnet publish -p PublishProfile=DefaultContainer -p ContainerBaseImage=mcr.microsoft.com/dotnet/aspnet:8.0-jammy-chiseled -p ContainerRegistry=docker.io -p ContainerRepository=<repo>/dockerdemoapp
	> ```

    This command instructs .NET to publish your app as a container directly to your Docker Hub repository.

2. **Verify**: Go to Docker Hub to verify that the image is uploaded there. No local image is created in this case.

### Using Apple Silicon

Apple´s new M* chips, also called _Apple Silicon_, are based on the ARM architecture. In order to build docker images on these Mac computers you need to specifically build the image for the AMD/Intel architecture.

In order to build an AMD based image, on Apple Silicon, you need to use the --platform argument like this:

```bash
docker build --platform linux/amd64 -t <repo>/dockerdemoapp .
```

In the case of using the two-step-approach for building the ASP.NET image we need to change the Dockerfile as well

In order for the first step to complete it needs to keep using an ARM based base image, while the second step should get an AMD based base image according to the --platform argument. So, we need to explicitly choose an ARM based base image for the first step in the Dockerfile. See below for a revised Dockerfile: (Note the difference in the first line)

> Dockerfile (for use on Apple Silicon)

```Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-jammy-arm64v8 AS build-env
WORKDIR /App

# Copy everything
COPY . ./
# Restore as distinct layers
RUN dotnet restore
# Build and publish a release
RUN dotnet publish -c Release -o out

# Build runtime image
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /App
COPY --from=build-env /App/out .
ENTRYPOINT ["dotnet", "DockerDemoApp.dll"]
```

To make the Dockerfile work across architecture we could modify it to use arguments in the docker build command like this:

> Dockerfile (with argument to change base image)

```Dockerfile
# Define an argument with a default value
ARG BASE_IMAGE_TAG=8.0

# Use the argument in the FROM instruction
FROM mcr.microsoft.com/dotnet/sdk:${BASE_IMAGE_TAG} AS build-env
WORKDIR /App

# Copy everything
COPY . ./
# Restore as distinct layers
RUN dotnet restore
# Build and publish a release
RUN dotnet publish -c Release -o out

# Build runtime image
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /App
COPY --from=build-env /App/out .
ENTRYPOINT ["dotnet", "DockerDemoApp.dll"]
```

Use the following command to change the base image:

```bash
docker build --build-arg BASE_IMAGE_TAG=8.0-jammy-arm64v8 --platform linux/amd64 -t <repo>/dockerdemoapp .
```

You can find base images in [Microsoft Artifact Registry](https://mcr.microsoft.com/en-us/product/dotnet/sdk/tags)

## Step 2: Provision the Container Host

Now that your app is containerized and pushed to a registry, the next step is to set up an environment on Azure where your container will run.

### Alternative 1: Azure VM (Ubuntu 22.04)

This section is based on this [Docker article](https://docs.docker.com/engine/install/ubuntu/)

For those who prefer a hands-on approach and full control, setting up an Azure VM as your Docker host might be the right choice.

1. **Create a Cloud-Init Configuration**:

    Save the following configuration in a file named `cloud-init_docker.yaml`. This script will install Docker on your Ubuntu VM.

	> cloud-init_docker.yaml

	```yaml
	#cloud-config

	# Install Docker Engine
	runcmd:
	# Add Docker's official GPG key
	- apt-get update -y
	- apt-get install ca-certificates curl -y
	- install -m 0755 -d /etc/apt/keyrings
	- curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
	- chmod a+r /etc/apt/keyrings/docker.asc

	# Add the repository to Apt sources
	- echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

	# Install Docker
	- apt-get update -y
	- apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
	```

2. **Provision Your VM with Azure CLI**:

    Execute the following script to create your VM and apply your Docker setup:

	> provision_vm.sh

	```bash
	#!/bin/bash

	resource_group=DockerDemoRG
	vm_name=DockerDemoVM
	vm_port=80

	az group create --location northeurope --name $resource_group

	az vm create --name $vm_name --resource-group $resource_group \
				--image Ubuntu2204 --size Standard_B1s \
				--generate-ssh-keys \
				--custom-data @cloud-init_docker.yaml

	az vm open-port --port $vm_port --resource-group $resource_group --name $vm_name
	```

    > Make sure to replace `<resource_group>`, `<vm_name>`, and `<vm_port>` with your specific values.

	Finally run the provisioning script:

	```bash
	./provision_vm.sh
	```

	> Note: Don't forget to `chmod +x provision_vm.sh` if you are on Linux or Mac


3. **Verify Installation**:

    SSH into your VM and run:

    ```bash
    sudo docker run hello-world
    ```

    This command verifies Docker is installed and running correctly.

### Alternative 2: Azure Container Apps (Serverless)

If you're looking for a simpler, serverless option, _Azure Container Apps_ might be a better choice. It is based on a Kubernetes cluster behind the scenes and one of its features is that it can scale down to zero container instances. This makes it a good alternative to keep costs down.

To deploy your application using Azure Container Apps, follow these steps:

1. **Create a Container App**:

	Basics

	1. **Access the Azure Portal**: Start by logging into your Azure account through the portal.
	2. **Navigate to Container Apps**: Look for the "Container Apps" option in the left sidebar. If it's not immediately visible, you can find it by using the search bar in the "All services" menu.
	3. **Initiate a New Container App**: Click the "+ Create" button to begin the setup of a new Container App.
	4. **Specify a Resource Group**: You'll need to either create a new resource group or select an existing one. Resource groups help organize your Azure resources.
	5. **Name Your App**: Assign a unique and descriptive name to your Container App.
	6. **Select a Region**: Choose an Azure region that is closest to your target audience for optimal performance.
	7. **Configure the Environment**: Either set up a new Container Apps Environment or select an existing one. This environment is where your app will live.

	Container
	
	8. **Opt Out of Quickstart Images**: Make sure to uncheck the "Use quickstart image" option to customize your deployment.
	9. **Select Your Registry**: Choose "Docker Hub" as the registry where your container image resides.
	10. **Specify Your Image**: Enter the name and tag of your Docker image in the "Image and tag" field (e.g., `<repo>/dockerdemoapp`).

	Ingress
	
	11. **Enable Ingress**: Turning on ingress allows external traffic to reach your app.
	12. **Traffic Configuration**: Set the app to accept traffic from anywhere, ensuring broad accessibility.
	13. **Client Certificate Settings**: Opt to ignore client certificates to simplify the setup.
	14. **Connection Security**: Permit insecure connections, though for production environments, you might want to revisit this setting.
	15. **Adjust the Target Port**: Set the target port to 8080, the default for ASP.NET applications, to direct traffic to your app.
	16. **Review and Launch**: After reviewing your settings, click "Review + create", and then "Create" to deploy your Container App.

2. **Verify Your Deployment**:

    Once deployed, navigate to the provided URL in your Container App's overview page to see your app live.

## Step 3: Deploy the Containerized App

With your hosting environment prepared and the image in a public repo on Docker Hub, it's time to deploy your app.

### Alternative 1: SSH into the Azure VM

For those who set up a VM, SSH into your machine and run the following command to start your app:

```bash
sudo docker run -d -p 80:8080 <repo>/dockerdemoapp
```

> Replace `<repo>` with your Docker Hub repository name. This command runs your container and maps port 80 on the VM to port 8080 in your container.

### Alternative 2: Azure CLI

For a hands-off deployment, use the Azure CLI to execute commands on your VM:

Create a file called `deploy.sh`

> deploy.sh

```bash
#!/bin/bash

resource_group=DockerDemoRG
vm_name=DockerDemoVM

# Run script on VM
az vm run-command invoke -g $resource_group -n $vm_name --command-id RunShellScript --scripts "docker run -d -p 80:8080 <repo>/dockerdemoapp"
```

Run the script:

```bash
./deploy.sh
```

> Note: Don't forget to `chmod +x deploy.sh` if you are on Linux or Mac

This script remotely instructs your VM to pull your Docker image from Docker Hub and run it, making your application accessible via the VM's public IP address.

### Verify Your App

After deploying your app:

1. **Find Your VM's IP Address**: Navigate to the Azure Portal, find your VM, and locate its public IP address.
2. **Access Your App**: Open a web browser and enter the public IP address. 

You should see your ASP.NET web application running, indicating a successful deployment.

## Troubleshooting

If you use Apple Silicon read the separate instructions carefully.

## Final Thoughts

This tutorial covered everything from creating a simple ASP.NET Core project to deploying it as a containerized application in Azure.

Containerization offers many benefits, like consistency across environments, scalability, and more efficient resource utilization. Azure provides several different services to host these containerized applications, where we have used two - An Ubuntu VM and an Azure Container App.

# Happy container! 🚀








