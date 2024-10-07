+++
title = 'Helm'
weight = 4
date = 2024-10-07
draft = false
+++

# Demo - Kubernetes Helm

### Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

**With Chocolatey (package manager for Windows)**


- https://chocolatey.org/install
- https://helm.sh/docs/intro/install/

> Run the following in PowerShell as Administrator

```ps
Set-ExecutionPolicy AllSigned

Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

choco install kubernetes-helm
```

Verify 

```bash
helm ls
```



### Deploy the app with Helm

1. Create a Helm Chart

	```bash
	helm create todo-app
	```

	- Delete all files in the `/templates` directory and copy all Kubernetes manifests to it

	```bash
	mongo-express-deployment.yaml
	mongo-express-service.yaml
	mongo-init-job.yaml
	mongodb-service.yaml
	mongodb-statefulset.yaml
	todo-configmap.yaml
	todo-deployment.yaml
	todo-service.yaml
	```

1. Install the chart as a release

	```bash
	helm install todo-app-release ./todo-app
	helm ls
	kubectl get all
	```

	- Verify by opening a browser and go to the todo app


1. Uninstall the release

	```bash
	helm uninstall todo-app-release
	helm ls
	kubectl get all
	```

## Deploy on GKE

1. Edit the Helm file `values.yaml`
	
	> `values.yaml`
	
	```yaml
	serviceType: NodePort
	```

1. Create another Helm file `values-gke.yaml`
	
	> `values-gke.yaml`

	```yaml
	serviceType: LoadBalancer
	```

1. Update the `type` in the template yaml-files to `{{ .Values.serviceType }}`
	
	>  `mongo-express-service.yaml`
	
	```yaml
	apiVersion: v1
	kind: Service
	metadata:
	  name: mongo-express-service
	spec:
	  selector:
	    app: mongo-express
	  ports:
	    - protocol: TCP
	      port: 8081
	      targetPort: 8081
	  type: {{ .Values.serviceType }}
  	```
		
	> `todo-service.yaml`
	
	```yaml
	apiVersion: v1
	kind: Service
	metadata:
	 name: todo-service
	spec:
	 selector:
	   app: todo-webapp
	 ports:
	 - protocol: TCP
	   port: 80
	   targetPort: 80
	 type: {{ .Values.serviceType }}
 	```	
	
3. Switch kubectl

	```bash
	kubectl config get-contexts
	kubectl config use-context <GKE>
	kubectl config current-context
	```

3. Install the Helm chart

	```bash
	helm install todo-app-release ./todo-app --values ./todo-app/values-gke.yaml
	helm ls
	kubectl get all
	```

1. Uninstall the release

	```bash
	helm uninstall todo-app-release
	helm ls
	kubectl get all
	```
