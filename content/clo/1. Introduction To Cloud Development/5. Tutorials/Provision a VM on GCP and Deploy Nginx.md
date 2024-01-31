+++
title = 'Provision a Virtual Machine on Google Cloud Platform and Deploy an Nginx Web Server'
weight = 2
date = 2024-01-23T11:28:13+01:00
draft = false
+++

## Introduction

This tutorial is designed for individuals with a basic understanding of cloud concepts and who are interested in learning how to set up a virtual machine (VM) on Google Cloud Platform (GCP) and deploy an Nginx web server. We will go step-by-step through the process, emphasizing best practices and troubleshooting.

## Method

- We will use the GCP web console to manually provision a VM running the Linux distribution Debian as Operating System (OS)
- We will deploy an Nginx web server by using the Debian package manager running commands in a terminal.
- We will use the web based terminal provided by the GCP Console to access the server

## Prerequisites

- A Google Cloud account. If you don't have one, sign up [here](https://cloud.google.com/).
- Basic knowledge of cloud computing and familiarity with command-line interfaces (CLI).

## Provision a Virtual Machine

1. Log into Google Cloud Console
	- Go to the [Google Cloud Console](https://console.cloud.google.com/).
	- Log in using your Google account.

2. Create a New Project (if you don't have one already)
	- In the console, go to the project selector page.
	- Click on "New Project" and enter a project name and billing details.

3. Creating the VM Instance
	- Navigate to the "Compute Engine" > "VM Instances".
	- Click "Create Instance".
	- Name your instance and select a region and zone.
	- Choose a machine type (E2 micro).
	- Under 'Boot disk', click 'Change', select the operating system (e.g., Debian) and click 'Select'.
	- Open port 80 for HTTP traffic
	- Click "Create" to provision the VM.

> - Most often you want to select the region closest to you
> - The instance type _E2 micro_ is cheap and sufficient for many test purposes
> - Debian is considered a very stable Linux distro and usually the default OS on GCP VMs
## Deploy Nginx

1. Accessing the VM
	- Once the VM is set up, click on "SSH" in the VM instances page to open a terminal in the browser.

2. Install Nginx
	- Update your package lists: `sudo apt-get update`
	- Install Nginx: `sudo apt-get install nginx`

## Verify the Installation

- Access your new server by browsing to the **external IP** of your VM, displayed in the VM instances page. You should see the Nginx welcome page.

If the you can't reach the welcome page above there are two more verification steps you can perform in order to pinpoint the error - most likely it is the firewall though.

1. Browse to the page using the terminal and the command: `curl localhost`. This command will output the page in raw HTML code to the terminal. This will work even if the firewall is closed since you browse to yourself within the same server.
2. If this doesn't work either you can make sure the nginx webserver is actually running by running the command: `sudo systemctl status nginx`. This will give you an output that indicates weather the service is active or not.

> In distros like Debian and Ubuntu services are usually started automatically after installation. However, there are many popular distros where this is not the case. Then you need to start and enable the service manually like so:
	- `sudo systemctl start nginx`
	- `sudo systemctl enable nginx`
## Additional Configuration (Optional)

- You can customize the served content from your Nginx server by editing the index file:
	`sudo nano /var/www/html/index...`

## Troubleshooting

- If you encounter issues accessing your server, ensure your VM's firewall rules allow HTTP traffic.
- Verify that Nginx is running: 
	`systemctl status nginx`

## Final Thoughts

Limitations: 
- The content is served unencrypted on HTTP
- The provisioning and deployment is done manually. No automation is applied.
- Access to the VM is done through the built-in web based terminal in the GCP console, which means you need access to the GCP platform in order to access the VM

# Happy cloud computing! 🚀