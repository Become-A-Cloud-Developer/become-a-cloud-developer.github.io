+++
title = 'Ingress'
weight = 6
date = 2024-10-15
draft = false
+++

# Demo - Kubernetes Ingress

### Preparations

Make sure your kubectl CLI is pointing to the cluster you want to work with:

```bash
kubectl config get-contexts
kubectl config delete-context <cluster>
kubectl config use-context <cluster>
```

Install your app and make sure that you don´t use `NodePort`or `LoadBalancer` to expose your services. Only use `ClusterIP`

I will use Helm to install the app in its own namespace

```bash
helm install todo-release ./TodoWithIngress --namespace todo --create-namespace
```

You can use port forwarding to verify. that the service you want to expose later works as expected

```bash
kubectl get ns
kubectl get all -n todo
kubectl port-forward service/todo-service 8080:80 -n todo
```

Command to create a cluster on GCP (adjust the values to your own account)

```bash
gcloud container --project "halogen-proxy-343613" clusters create-auto "autopilot-cluster-ingressdemo" --region "europe-north1" --release-channel "regular" --network "projects/halogen-proxy-343613/global/networks/default" --subnetwork "projects/halogen-proxy-343613/regions/europe-north1/subnetworks/default" --cluster-ipv4-cidr "/17"
```

### Deploy the Nginx Ingress Controller
https://kubernetes.github.io/ingress-nginx/deploy/#quick-start

1. Install the **Nginx Ingress Controller**:
  
	```bash
	helm upgrade --install ingress-nginx ingress-nginx \
	  --repo https://kubernetes.github.io/ingress-nginx \
	  --namespace ingress-nginx --create-namespace
	```
	or
	
	```bash
	kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.4/deploy/static/provider/cloud/deploy.yaml
	```
	
	By default the script will use `LoadBalancer`. You can change this to `NodePort` by running the following patch:
	
	```bash
	kubectl get ns
	kubectl get all -n ingress-nginx
	kubectl patch service/ingress-nginx-controller -n ingress-nginx -p '{"spec": {"type": "NodePort"}}'
	kubectl get all -n ingress-nginx
	```
	
2. Create an Ingress Resource:
	
	Based on the URL we want to direct traffic to different services. One that routes traffic to your ASP.NET Core webapp and another to the Mongo Express UI
	
	- webapp.mydomain.tld -> todo app
	- mongo-express.mydomain.tld -> mongo express
	
	> Make sure the namespace is correct!
	
	Create a new file `ingress.yaml` 
	
	> `ingress.yaml`
	
	
	```yaml
	apiVersion: networking.k8s.io/v1
	kind: Ingress
	metadata:
	  name: my-ingress
	  namespace: todo
	  annotations:
	    kubernetes.io/ingress.class: "nginx"
	spec:
	  rules:
	  - host: webapp.mydomain.tld
	    http:
	      paths:
	      - path: /
	        pathType: Prefix
	        backend:
	          service:
	            name: todo-service
	            port:
	              number: 80
	  - host: mongo-express.mydomain.tld
	    http:
	      paths:
	      - path: /
	        pathType: Prefix
	        backend:
	          service:
	            name: mongo-express-service
	            port:
	              number: 8081
	```
	
	Adjust the services to your own service names. Also, replace the `host` fields with your desired domain/subdomain names.
	
	Apply the Ingress:
	
	```bash
	kubectl apply -f ingress.yaml
	```
	
### Point Your Domain Names

For the Ingress resource to work, you'll need to point `webapp.mydomain.tld` and `mongo-express.mydomain.tld` to your Kubernetes cluster:

- **On Docker Desktop:** You can add these domains to your local machine's `hosts` file pointing to `127.0.0.1`.

	> Mac
	
	```bash
	sudo nano /etc/hosts
	```
	
	```bash
	% cat /etc/hosts
	##
	# Host Database
	#
	# localhost is used to configure the loopback interface
	# when the system is booting.  Do not change this entry.
	##
	127.0.0.1       localhost
	255.255.255.255 broadcasthost
	::1             localhost
	# Added by Docker Desktop
	# To allow the same kube context to work on the host and the container:
	127.0.0.1 kubernetes.docker.internal
	# End of section
	
	127.0.0.1       webapp.mydomain.tld
	127.0.0.1       mongo-express.mydomain.tld
	```

	> Windows
	
	- Open Notepad with "Run as administrator"
	- Open the file `C:\Windows\System32\drivers\etc\hosts`
		- In Notepad, go to File > Open.
		- Navigate to `C:\Windows\System32\drivers\etc`.
		- Change the file type dropdown from "Text Documents (.txt)" to "All Files (.*)".
		- Select the `hosts` file and click Open.
	- Flush DNS (optional, but sometimes needed):
		- Open Command Prompt as Administrator.
		- Type `ipconfig /flushdns` and press Enter.

- **On GKE:** You would point these domains/subdomains to the external IP address of your Ingress. You can find this by running:

```bash
kubectl get ingress my-ingress -n todo
```

### Verify

Run the following command to get the port on which the nginx controller is exposed

```bash
kubectl get service/ingress-nginx-controller -n ingress-nginx
```


Verify by opening a browser (make sure to include http://) and go to:

- `http://webapp.mydomain.tld:<port>`
- `http://mongo-express.mydomain.tld:<port>`



