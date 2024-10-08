+++
title = 'Kubernetes GKE'
weight = 3
date = 2024-10-07
draft = false
+++

# Demo - Kubernetes GKE

### Install `gcloud` CLI

<details>
<summary> test</summary>
random test text
</details>

Before deploying to GKE, you need to set up `gcloud` (the Google Cloud SDK command-line tool)

1. Install python 3
1. Install the [Google Cloud SDK](https://cloud.google.com/sdk/docs/install).

### Prepare `kubectl` for GKE

https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke

install the gke-gcloud-auth-plugin

`gcloud components install gke-gcloud-auth-plugin`

> Note: gcloud is the recommended way to install the binary on Windows and OS X
> 
> You need to have the CLI for GCP `gcloud` installed

Verify

`gke-gcloud-auth-plugin --version`

For Windows

`gke-gcloud-auth-plugin.exe --version`

**Login to GCP from gcloud CLI**

`gcloud auth login`

> **Switching between kubectl contexts**
> 
> If you're using Docker Desktop's Kubernetes cluster on macOS or Windows, and you've switched your `kubectl` context to another cluster (like GKE), you can switch it back to Docker Desktop's cluster with the following steps:
> 
> 1. First, you can list your available contexts with:
> 
> 
>     ```bash
>     kubectl config get-contexts
>     ```
> 
> 2. You'll see a list of available contexts. The Docker Desktop cluster typically has the context name `docker-desktop` or `docker-for-desktop`.
> 
> 3. To switch to the Docker Desktop context, use:
> 
> 
>     ```bash
>     kubectl config use-context docker-desktop
>     ```
>     or, if the context is named differently:
>     
>     ```bash
>     kubectl config use-context docker-for-desktop
>     ```
> 
> 4. Verify that you've switched to the Docker Desktop context:
> 
> 
>     ```bash
>     kubectl config current-context
>     ```
> 
> This command should now return `docker-desktop`, indicating you're now working with your Docker Desktop Kubernetes cluster.
> 
> 

### **Step 1: Setup K8S Cluster**

2. Go to the GCP console and choose `Kubernetes Engine`

   - Go to Clusters
   - Create (AutoPilot)
   - Connect

4. **Authenticate kubectl with the cluster**:

   After your cluster is created, you'll need to get its credentials to configure `kubectl`:

   ```bash
   gcloud container clusters get-credentials <cluster> --region <region> --project <project>
   ```
4. Verify that you've switched to the GKE context:
 
   ```bash
   kubectl config current-context
   ```

### **Step 2: Deploying the nginx Deployment**

Using the deployment YAML file `nginx-deployment.yaml`:

> `nginx-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
```

Run the following command:

```bash
kubectl apply -f nginx-deployment.yaml
```

Check the status of the deployment:

```bash
kubectl get deployments
```

### **Step 3: Exposing the nginx Deployment using a Service**

Using the service YAML file `nginx-service.yaml`:

> `nginx-service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: NodePort
```

Run the following command:

```bash
kubectl apply -f nginx-service.yaml
```

Check the status of the service:

```bash
kubectl get svc
```

This will show you the `NodePort` on which the `nginx` service is exposed. 

Unfortunately the service has no external IP making it unaccessible. But there is a way to go directly to a pod with the port-forwarding function in kubectl.

```bash
kubectl port-forward service/nginx-service 8080:80

Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
```
Open a browser and go to `localhost:8080`


Change the service type to `LoadBalancer` and re-apply the service. Verify that a loadbalancer is created

```bash
kubectl apply -f nginx-service.yaml
kubectl get svc
```
Open a browser and go to `<Eternal IP>`



> ### **Warning**
> 
> Don´t forget to delete
> 
> 1. Kubernetes cluster
> 1. Loadbalancer
