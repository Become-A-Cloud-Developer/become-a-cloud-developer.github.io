+++
title = "Provision a Virtual Private Cloud on GCP And Create A Firewall Rule"
weight = 4
date = 2024-02-05
draft = false
+++

## Introduction

This tutorial is designed for individuals with a basic understanding of cloud concepts and who are interested in learning how to set up a Virtual Private Cloud (VPC) on Google Cloud Platform (GCP) and create a firewall rule that opens port 8080 for Internet traffic. We will use an nginx web server listening on port 8080 to verify the solution. We will go step-by-step through the process, emphasizing best practices and troubleshooting.

## Method

- We will use the GCP web console to manually provision a VPC
- We will create a firewall rule that opens port 8080 for Internet traffic
- We will use an nginx web server that listens to port 8080 in order to verify the solution

## Prerequisites

- A Google Cloud account. If you don't have one, sign up [here](https://cloud.google.com/).
- Basic knowledge of cloud computing and familiarity with the GCP web console.

## Create a VPC Network in GCP

1. Log into Google Cloud Console
	- Go to the [Google Cloud Console](https://console.cloud.google.com/).
	- Log in using your Google account.

2. Create a New Project (if you don't have one already)
	- In the console, go to the project selector page.
	- Click on "New Project" and enter a project name and billing details.

3. Navigate to VPC networks
	- With your new project selected, navigate to the "Navigation Menu" (three horizontal lines in the top left corner).
	- Scroll down and select "VPC network" > "VPC networks"

3. **Create VPC network**
	- Click the "Create VPC network" button.
	- **Name**: Enter a name for your VPC network. (demonet)
	- **Subnets**: Select "Custom" to manually specify your subnet settings.
	- Under "New subnet":
		- enter a subnet name. (demosubnet)
		- specify the region that matches your intended VM's location.
		- enter the desired IP address range in CIDR format. (10.0.0.0/24)
		- Click "Done"
	- Select all default firewall rules
	- Click "Create" to finalize the VPC network creation.
	
## Set Up a Firewall Rule to Open Port 8080

1. Select the newly created VPC network
	- Select the "Firewalls" tab

> It is possible to get to the firewall rules via the main side menu -  "VPC network" > "Firewall" - as well

2. **Create a Firewall Rule**
	- Click the "Add Firewall Rule" button.
	- **Name**: Enter a name for your firewall rule. (e.g allow-8080)
	- **Targets**: Select "Specified target tags".
	- **Target tags**: Enter the tag you used for your VM (e.g., `allow-8080`).
	- **Source filter**: Select "IPv4 ranges".
	- **Source IP ranges**: Enter `0.0.0.0/0` to allow access from anywhere on the internet.
	- **Protocols and ports**: Specify "tcp" and enter `8080` for the port..
	- Click "Create".

## Create a Virtual Machine with Nginx To Verify the Firewall

1. Creating the VM Instance
	- Navigate to the "Compute Engine" > "VM Instances".
	- Click "Create Instance".
	- Name your instance and select a region and zone.
	- Choose a machine type (E2 micro).
	- Make sure the OS is Debian
	- **Do NOT open the firewall here!**
	- Scroll down and expand the "Advanced options" - "Networking".
		- Change the "Network interface" (demonet, demosubnet)
		- Click "Done"
	- Click "Create" to provision the VM.

> Note that you can't click the external IP address

1. Install Nginx
	- After the VM is created, SSH into it from the VM instances page by clicking the "SSH" button next to your VM.
	- Run these commands to install nginx:
		```bash
		sudo apt-get update -y
		sudo apt-get install nginx -y
		```
	- Verify the installation by running (you should see the HTML payload):
		```bash
		curl localhost
		```

3. Set Up Nginx to Listen on Port 8080

	- Edit the configuration file `/etc/nginx/sites-available/default`
		```bash
		sudo nano /etc/nginx/sites-available/default
		```
	- Find the line that says `listen 80 default_server;` and change `80` to `8080`		```
	- Save and close the file.
	- **Restart Nginx** to apply the changes: `sudo systemctl restart nginx`
		```bash
		sudo systemctl reload nginx
		```
	- Verify the installation by running (you should see the HTML payload):
		```bash
		curl localhost:8080
		```

4. Add network tag to the VM in order to apply the firewall rule
	- Select the newly created VM
	- Click the "Edit" button
	- Scroll down to the "Network interfaces" section
		- Add network tag (allow-8080)
	- Click "Save"

## Verify Everything Works

- **Check Nginx**: In your web browser, visit `http://[VM_EXTERNAL_IP]:8080`. You should see the Nginx welcome page.

## Troubleshooting

- Make sure your VM is attached to your VPC and not to the default VPC
- Check that the network tag is correct
- Make sure you add the port to the IP when browsing (IP:Port - 35.228.225.203:8080)
## Final Thoughts

- It's usually not a good idea to expose a web app directly to the internet. Use a load balancer or reverse proxy to face the Internet instead.

## Don't Forget

Google Cloud Storage is a paid service, and charges apply for storage, network usage, and other features like operations and retrieval. Make sure to review the current pricing and manage your resources accordingly.

Don't forget to delete all resources once you are done with this tutorial.
- Virtual Machine
- VPC

# Happy cloud computing! 🚀