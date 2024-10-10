+++
title = 'ArgoCD'
weight = 5
date = 2024-10-10
draft = false
+++

# Demo - Kubernetes ArgoCD


### Install ArgoCD

1. Install Argo CD in the Kubernetes cluster
	
	`https://argo-cd.readthedocs.io/en/stable/getting_started?_gl=1*j1x42g*_ga*MTk3MjE0ODg5MS4xNjk0NjAyMDk3*_ga_5Z1VTPDL73*MTY5NDYwMjEwNS4xLjAuMTY5NDYwMjEwOS4wLjAuMA..`
	
	```bash
	kubectl create namespace argocd
	kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
	```
	
	This will create a new namespace, argocd, where Argo CD services and application resources will live.
	
2. Install Argo CD CLI
	
	`https://argo-cd.readthedocs.io/en/stable/cli_installation/`
	
3. Access The Argo CD API Server

	By default, the Argo CD API server is not exposed with an external IP. To access the API server, choose one of the following techniques to expose the Argo CD API server:
	
	**Service Type: Load Balancer**
	
	Change the argocd-server service type to LoadBalancer:
	
	```bash
	kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
	```
	
	**Port Forwarding**
	
	Kubectl port-forwarding can also be used to connect to the API server without exposing the service.
	
	```bash
	kubectl port-forward svc/argocd-server -n argocd 8080:443
	```
	
	The API server can then be accessed using https://localhost:8080
	
4. Login Using The CLI

	The initial password for the admin account is auto-generated and stored as clear text in the field password in a secret named `argocd-initial-admin-secret` in your Argo CD installation namespace. You can simply retrieve this password using the argocd CLI:
	
	```bash
	argocd admin initial-password -n argocd
	```
	
	Using the username _admin_ and the password from above, login to Argo CD's IP or hostname:
	
	```bash
	argocd login <ARGOCD_SERVER>
	```
	
	Change the password using the command:
	
	```bash
	argocd account update-password
	```


### Setup the Git repo

1. Create a git repo

> `https://github.com/larsappel/ArgoCD.git`

### Create App in ArgoCD

1. Point to git repo
2. Sync

## Github Action

> Enable the workflow action to update the repo:
> 
> (Repo) Settings > Actions > General > Workflow permissions > Read and write permissions > Save
> 
> Don´t forget to make the Github Package registry public 
> 
> You need a PAT Personal Access Token (classic) set in secret GH_TOKEN inn the workflow below

Put the app and the manifests in the same repo but in different directories:

- ToDoApp (including the Dockerfile)
- ToDoApp-K8S-Manifests

1. The github action will be triggered by a change in the application
1. Build a new Docker image and upload to the **Github Package** registry
2. Update the manifest with the new image version

	> build-push-image.yaml
	
	```bash
		
	name: Build and Push Docker Image
	
	on:
	  push:
	    branches:
	      - main
	    paths:
	    - 'ToDoApp/**'
	  workflow_dispatch:
		
	jobs:
	  build:
	    runs-on: ubuntu-latest
		
	    steps:
	    # Checkout the repository
	    - name: Checkout code
	      uses: actions/checkout@v4
		
	    # Login to GitHub Container Registry
	    - name: Login to GitHub Container Registry
	      uses: docker/login-action@v1
	      with:
	        registry: ghcr.io
	        username: ${{ github.actor }}
	        password: ${{ secrets.GH_TOKEN }}
		
	    # Build and push the Docker image
	    - name: Build and Push Docker image
	      run: |
	        cd ToDoApp
	        docker build -t ghcr.io/larsappel/todoapp:${{ github.sha }} .
	        docker push ghcr.io/larsappel/todoapp:${{ github.sha }}
		
	    # Update the Kubernetes manifests to use the new image tag
	    - name: Update Kubernetes manifests
	      run: |
	        sed -i 's|ghcr.io/larsappel/todoapp:.*|ghcr.io/larsappel/todoapp:${{ github.sha }}|' ToDoApp-K8S-Manifests/todo-deployment.yaml
		
	    # Commit and push the updated manifests back to the repository
	    - name: Commit and push updated manifests
	      run: |
	        git config --global user.name "GitHub Actions"
	        git config --global user.email "github-actions@github.com"
	        git add ToDoApp-K8S-Manifests/todo-deployment.yaml
	        git commit -m "Update webapp image to ${{ github.sha }}"
	        git push
	```

4. Continue to create an App in ArgoCD as before

> Use portforwarding to open ArgoCD at https://localhost:<port>/login
> 
> Open in a private browser window if you encounter problems




## Deploy on GKE

Prerequisites

Install Lens

1. Install ArgoCD with Helm in Lens
2. Go to secrets and check the password
3. Port forward the argocd-server and browse to thee UI (user: admin / password: <from secrets>
4. Create App in ArgoCD as before


