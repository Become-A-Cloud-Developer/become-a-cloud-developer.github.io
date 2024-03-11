+++
title = "Use Github Actions to Deploy your App"
weight = 8
date = 2024-03-11
draft = false
+++

## Introduction

In this guide, we'll walk you through how to use Github Actions to deploy an app to Azure VM

## Method

- Prepare Github repo and add Github workflow
- Implement an example app
- Develop a CI/CD pipeline in the workflow starting with a github hosted runner
- Provision a VM with a self-hosted gihub runner 
- Deploy the app using github actions - triggered by push and manually

## Prerequisites

- An Azure account. If you don't have one, sign up at [Azure's official site](https://azure.microsoft.com/).
- Basic familiarity with Azure services and ASP.NET web development.
- Github account
- Basic familiarity with git and Github
- VSCode connected to Github

## Step 1: Create a new repo on Github containing an ASP.NET webapp

The end point of the developing effort and the start point of a CI/CD pipeline is many times a git repository. Let's first setup a git repo and "develop" a webapp, which we check-in to the repo. 

1. Create a new directory `GithubActionsDemo`
2. Open the directory in VSCode and create a new ASP.NET webapp

    ```bash
    dotnet new webapp
    dotnet new gitignore
    ```

3. Initiate a git repo and push it to Github

    ```bash
    git init
    git add .
    git commit -m "Initial Commit"
    ```

4. Use the built in git function in VSCode to push the repo to Github

## Step 2: Create a Github Workflow with a github-hosted runner

We now have our webapp checked-in. The first step is to automate the CI part in the CI/CD pipeline. In this step we will build and publish the webapp to an _Artifact Repository_. We will in this tutorial use the one hosted on Github.

1. Login to Github and navigate to your newly created repo `GithubActionsDemo`.
2. Select the tab **Actions** and search for **.net** in the search bar.
3. Click **Configure** in the **.NET** workflow
4. Edit the workflow:

> cicd.yaml

```yaml
# This workflow will build the GithubActionsDemo project

name: GithubActionsDemo

on:
  push:
    branches:
    - "main"
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:

    - name: Install .NET SDK
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '8.0.x'

    - name: Check out this repo
      uses: actions/checkout@v4

    - name: Restore dependencies (install Nuget packages)
      run: dotnet restore

    - name: Build and publish the app
      run: |
        dotnet build --no-restore
        dotnet publish -c Release -o ./publish

    - name: Upload app artifacts to Github
      uses: actions/upload-artifact@v4
      with:
        name: app-artifacts
        path: ./publish

```

5. Name the file `cicd.yaml` and commit the change into the repo by pressing the button **Commit changes...**.
    - Click **Commit changes** again in the popup window

### Verify the build step

In order to verify that the workflow works correctly, go the the **Actions** tab and select the running workflow.

Go to the workflow **Summary** side menu and verify that your artifacts are uploaded under the _Artifcts_ section. Download the artifacts and check the content.

If the build step went well you should see a checkmark in green.

> Pull the repo in VSCode before you continue developing the app - other wise you can get synch problems between your local repo and the one on Github.


## Step 3: Provision a VM with a self-hosted runner

The next step is to deploy the app to an Azure Linux VM. In order to do so, we first need to carry out some steps:

- Provision a Linux VM
    - install .Net Runtime
    - configure the VM (like add a service file)

- Develop a _deploy_ step for the workflow (the CD part in the CI/CD pipeline)

- Install a self-hosted runner on the VM (the Github runner will fetch new artifacts from Github)

### Provision a Linux VM

Create the following two files:

- provision_vm.sh
- cloud-init_dotnet.yaml

> provision_vm.sh

```bash
#!/bin/bash

resource_group=GithubActionsDemoRG
vm_name=GithubActionsDemoVM
vm_port=5000

az group create --location northeurope --name $resource_group

az vm create --name $vm_name --resource-group $resource_group \
             --image Ubuntu2204 --size Standard_B1s \
             --generate-ssh-keys --admin-username azureuser \
             --custom-data @cloud-init_dotnet.yaml

az vm open-port --port $vm_port --resource-group $resource_group --name $vm_name
```


> cloud-init_dotnet.yaml

```yaml
#cloud-config

# Install .Net Runtime 8.0
runcmd:
  # Register Microsoft repository (which includes .Net Runtime 8.0 package)
  - wget -q https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
  - dpkg -i packages-microsoft-prod.deb

  # Install .Net Runtime 8.0
  - apt-get update
  - apt-get install -y aspnetcore-runtime-8.0

# Create a service for the application
write_files:
  - path: /etc/systemd/system/GithubActionsDemo.service
    content: |
      [Unit]
      Description=ASP.NET Web App running on Ubuntu

      [Service]
      WorkingDirectory=/opt/GithubActionsDemo
      ExecStart=/usr/bin/dotnet /opt/GithubActionsDemo/GithubActionsDemo.dll
      Restart=always
      RestartSec=10
      KillSignal=SIGINT
      SyslogIdentifier=GithubActionsDemo
      User=www-data
      Environment=ASPNETCORE_ENVIRONMENT=Production
      Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false
      Environment="ASPNETCORE_URLS=http://*:5000"

      [Install]
      WantedBy=multi-user.target
    owner: root:root
    permissions: '0644'

systemd:
  units:
    - name: GithubActionsDemo.service
      enabled: true
```

chmod and run the shell script

```bash
chmod +x provision_vm.sh
./provision_vm.sh
```

#### Verify the provisioning

Go into the Azure portal and verify that a VM has been provisioned.

Login to the VM

```bash
ssh azureuser@<public_ip>
```

If you can login you are all set for the next step. You should see the following prompt:

```bash
azureuser@GithubActionsDemoVM:~$
```



### Develop a _deploy_ step for the workflow

We need to add a new _deploy_ job to the Github workflow file. Go to VSCode and open up the file `cicd.yaml`. Add the code below as a second job. (Make sure the indentation is correct!)

> cicd.yaml

```yaml

...

  deploy:
    runs-on: self-hosted
    needs: build

    steps:
    - name: Download the artifacts from Github (from the build job)
      uses: actions/download-artifact@v4
      with:
        name: app-artifacts

    - name: Stop the application service
      run: |
        sudo systemctl stop GithubActionsDemo.service
    
    - name: Deploy the the application
      run: |
        sudo rm -Rf /opt/GithubActionsDemo || true
        sudo cp -r /home/azureuser/actions-runner/_work/GithubActionsDemo/GithubActionsDemo/ /opt/GithubActionsDemo
      
    - name: Start the application service
      run: |
        sudo systemctl start GithubActionsDemo.service
```

Don't push the new workflow just yet. First we should install the runner.


### Install a self-hosted runner on the VM

1. Go to the Github repo:
    - select the **Settings** tab
    - Select **Actions -> Runners** in the side menu
    - Click **New self-hosted runner**
    - Select **Linux** (Architecture: x64)

2. Login to the VM and run the code
    - Press `<Enter>`to accept all default values in the configuration wizard

```bash
# The code from Github

...

./run.sh
```

If the installation process went well you should see the following output in the terminal

> Output

```bash
azureuser@GithubActionsDemoVM:~/actions-runner$ ./run.sh

√ Connected to GitHub

Current runner version: '2.314.1'
2024-03-11 13:39:55Z: Listening for Jobs
```

In the Github portal, you should also see the runner as _Idle_.

Now it's time to push your new Github workflow (the `cicd.yaml` file) and see how the runner picks up the job and carries it out.

### Verify that the deployment went well

After pushing the new code to Github you should see how the runner picks up the job.

```
azureuser@GithubActionsDemoVM:~/actions-runner$ ./run.sh

√ Connected to GitHub

Current runner version: '2.314.1'
2024-03-11 13:39:55Z: Listening for Jobs
2024-03-11 13:52:45Z: Running job: deploy
2024-03-11 13:52:57Z: Job deploy completed with result: Succeeded
```


Open up a browser and navigate to

``` 
http://<public_ip>:5000
```

You should now see your app running on the VM.


> ### Run the runner as a service instead
> 
> In a real scenario you should run the runner as a service in the background. You can do this by running the following commands:
> 
> ```bash
> # Run the Github runner as a service
> 
> sudo ./svc.sh install azureuser
> sudo ./svc.sh start
> ```
> 
> You can follow the logs with (use autocomplete with `<tab>` to follow your service)
> 
> ```bash
> sudo journalctl -u actions.runner.<user>-GithubActionsDemo.GithubActionsDemoVM.service -f
> ```

## Step 4: Use the new CI/CD Pipeline

1. Open up your project in VSCode and do some changes to your code.
2. Push the changes to the Github repo
3. Verify the changes in the browser (it will take a minute or two)

## Conclusion

This tutorial has walked you through the process of ...

## Don't Forget

Azure services incur costs. Delete resources you no longer need.

# Happy Deploying! 🚀
