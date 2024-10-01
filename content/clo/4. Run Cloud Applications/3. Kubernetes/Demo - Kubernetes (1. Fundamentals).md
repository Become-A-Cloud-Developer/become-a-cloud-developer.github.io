+++
title = 'Fundamentals'
weight = 1
date = 2024-10-01
draft = false
+++

# Demo - Kubernetes Fundamentals

## Preparation

There are many ways to install a local kubernetes cluster. In this demo we will use the built-in kubernetes function of Docker Desktop.

1. Install Docker Desktop

	- Enable the Kubernetes function in Docker Desktop

2. Open up your IDE (VSCode) in a new directory `Kubernetes`

	- Open the integrated terminal
	- Verify that KubeCtl works `kubectl version`

## Run an Nginx container in a local K8S cluster (only one node)

### Imperative commands

1. Deploy an nginx Pod:

	```bash
	kubectl run my-nginx-pod --image=nginx
	kubectl get pods
	```

1. Expose the nginx Pod as a service:

	```bash
	kubectl expose pod my-nginx-pod --type=NodePort --port=80
	kubectl get services
	```

1. Verify in the browser that it works or with `curl localhost:<PORT>`. You get the port number from the `kubectl get services` command above.

1. Clean up the resources (optional):

	```bash
	kubectl delete pod my-nginx-pod
	kubectl delete svc my-nginx-pod
	```

> ### NodePort

> In Kubernetes, a NodePort is one of the ways to expose your services to **external** traffic. When you define a service with type=NodePort, Kubernetes does the following:

> 1. Allocates a Port: Kubernetes allocates a port from a range (default is 30000-32767) for your service. This port is open on every node in your cluster.
> 1. Routes Incoming Traffic: Any traffic that hits a node on the allocated port is routed to the service. This works even if the actual pod backing the service isn't on the node that received the traffic.
> 1. Balancing: Even if multiple nodes in your cluster receive traffic on the NodePort, Kubernetes internally load balances that traffic to pods that are part of the service.

> Example:

> We exposed the pod `my-nginx-pod` as a service with the same name. When you expose the service as a _NodePort_ service, Kubernetes will allocate a port, for example, 30142 (from the default range).

> 1. Inside the cluster: The service is accessible at `<cluster-ip>:80` (ClusterIP is assigned to the service)
> 1. Outside the cluster: The service is accessible at `<node-ip>:30142` on any node in the cluster.

> ```bash
> $ kubectl get services
> NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
> kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP        55d>my-nginx-pod
> NodePort    10.103.104.6     <none>        80:30142/TCP   18m
> ```

### Declarative files

1. Deploy an nginx Pod:

	- Create a file `nginx-pod.yaml` and declare the pod in it

	> `nginx-pod.yaml`
	
	```bash
	apiVersion: v1          # The version of the Kubernetes API
	kind: Pod               # The kind of Kubernetes object; a Pod
	metadata:               # Metadata about the object
	  name: nginx-pod       # The name of the Pod; used to identify the Pod in commands and within the cluster
	  labels:               # Key-value pairs attached to the object; used for selection and organization
	    app: nginx          # Label with key "app" and value "nginx"; used by other objects like services to select this Pod
	spec:                   # Specification of the desired behavior of this Pod
	  containers:           # The list of containers that will be started in this Pod
	  - name: nginx         # The name of the container; must be unique within the Pod
	    image: nginx:latest # The container image to use; in this case, the latest version of the nginx image
	```

	- Apply the file to the cluster and check the result

	```bash
	kubectl apply -f nginx-pod.yaml
	kubectl get pod nginx-pod
	```
	


1. Expose the nginx Pod as a service:

	- Create a file `nginx-service.yaml` and declare the service in it

	> `nginx-service.yaml `
	
	```bash
	apiVersion: v1              # The version of the Kubernetes API
	kind: Service               # The kind of Kubernetes object; a Service
	metadata:                   # Metadata about the object
	  name: nginx-service       # The name of the Service; used to identify the Service in commands and within the cluster
	spec:                       # Specification of the desired behavior of this Service
	  selector:                 # Defines how the Service finds which Pods to route traffic to
	    app: nginx              # Selects any Pod with a label having key "app" and value "nginx"
	  ports:                    # The list of ports on which the Service listens and forwards traffic 
	  - protocol: TCP           # The network protocol this port listens to; in this case, TCP
	    port: 80                # The port on which the Service is exposed within the cluster
	    targetPort: 80          # The port on the Pod to which the traffic is directed
	  type: NodePort            # Specifies the type of Service; NodePort means it'll be exposed on each node's IP at a static port in the range 30000-32767
	```

	- Apply the file to the cluster and check the result

	```bash
	kubectl apply -f nginx-service.yaml
	kubectl get svc nginx-service
	```
	

1. Verify in the browser that it works or with `curl localhost:<PORT>`

1. Clean up the resources (optional):

	```bash
	kubectl delete -f nginx-pod.yaml
	kubectl delete -f nginx-service.yaml
	```
	or 
	
	```bash
	kubectl delete pod nginx-pod
	kubectl delete svc nginx-service
	```

## Deploy a ReplicaSet to ensure multiple replicas of an Nginx container are running


### Declarative files

1. Deploy an nginx ReplicaSet:

	- Create a file `nginx-rs.yaml` and declare the ReplicaSet in it

	> `nginx-rs.yaml`
	
	```bash
	apiVersion: apps/v1         # The version of the Kubernetes API used for ReplicaSets
	kind: ReplicaSet            # The kind of Kubernetes object; a ReplicaSet
	metadata:                   # Metadata about the object
	  name: my-nginx-rs         # The name of the ReplicaSet; used to identify it in commands and within the cluster
	  labels:                   # Key-value pairs attached to the object; used for selection and organization
	    app: nginx              # Label with key "app" and value "nginx"; can be used by other objects to select this ReplicaSet
	spec:                       # Specification of the desired behavior of this ReplicaSet
	  replicas: 2               # The number of pod replicas desired; ensures 2 pods with the specified configuration are always running
	  selector:                 # Defines how the ReplicaSet selects the pods it manages
	    matchLabels:            # Selection based on labels
	      app: nginx            # Selects any Pod with a label having key "app" and value "nginx"
	  template:                 # Template for the pods that will be created; defines how each pod should look
	    metadata:               # Metadata about the pods
	      labels:               # Labels to be added to each pod
	        app: nginx          # Each pod will have this label which will match the selector of the ReplicaSet
	    spec:                   # Specification of the desired behavior of the pods
	      containers:           # The list of containers that will be started in each pod
	      - name: nginx         # Name of the container within the pod; must be unique within the pod
	        image: nginx:latest # The container image to use; in this case, the latest version of nginx
	```

	- Apply the file to the cluster and check the result

	```bash
	kubectl apply -f nginx-rs.yaml
	kubectl get replicaset my-nginx-rs
	```

1. Verify that the replicas are picked up by the service
	- Run `kubectl describe svc nginx-service` and check the **`Endpoints:`** row

	```bash
	$ kubectl describe svc nginx-service
	Name:                     nginx-service
	Namespace:                default
	Labels:                   <none>
	Annotations:              <none>
	Selector:                 app=nginx
	Type:                     NodePort
	IP Family Policy:         SingleStack
	IP Families:              IPv4
	IP:                       10.104.104.248
	IPs:                      10.104.104.248
	LoadBalancer Ingress:     localhost
	Port:                     <unset>  80/TCP
	TargetPort:               80/TCP
	NodePort:                 <unset>  31764/TCP
	Endpoints:                10.1.0.49:80,10.1.0.50:80
	Session Affinity:         None
	External Traffic Policy:  Cluster
	Events:                   <none>
	```

1. Clean up the resources (optional):

	```bash
	kubectl delete -f nginx-rs.yaml
	```
	or 

	```bash
	kubectl delete replicasets
	```

> ### Labels

> Labels in Kubernetes are key-value pairs associated with objects, like Pods, Services, and Deployments. They serve two primary functions:

> 1. **Identification**: Labels help in organizing and selecting subsets of resources based on user-defined criteria. For instance, you might label all your production Pods with `env=production` and staging Pods with `env=staging`.

> 2. **Selection**: Many Kubernetes components, such as Services and Deployments, use labels to determine which Pods they apply to. For example, a Service might select all Pods with the label `app=nginx` to route network traffic to them.



## Run an Nginx container in a local K8S cluster using Deployments

### Imperative commands

1. Deploy an nginx Deployment:

    ```bash
    kubectl create deployment my-nginx-deployment --image=nginx --replicas=2
    kubectl get deployments
    ```

1. Expose the nginx Deployment as a service:

    ```bash
    kubectl expose deployment my-nginx-deployment --type=NodePort --port=80
    kubectl get services
    ```

1. Verify in the browser that it works or with `curl localhost:<PORT>`

1. Clean up the resources:

    ```bash
    kubectl delete deployment my-nginx-deployment
    kubectl delete svc my-nginx-deployment
    ```

### Declarative files

1. Deploy an nginx Deployment:

    - Create a file `nginx-deployment.yaml` and declare the deployment in it

    > `nginx-deployment.yaml`
    
    ```yaml
    apiVersion: apps/v1          # The version of the Kubernetes API for deployments
    kind: Deployment             # The kind of Kubernetes object; a Deployment
    metadata:                    # Metadata about the object
      name: my-nginx-deployment  # The name of the Deployment; used to identify it
    spec:                        # Specification of the desired behavior of this Deployment
      replicas: 2                # Desired number of pod instances
      selector:                  # Label selector for the pods
        matchLabels:
          app: nginx
      template:                  # The pod template to use
        metadata:
          labels:
            app: nginx           # Labels for pod selection
        spec:
          containers:
          - name: nginx          # The name of the container; must be unique within the Pod
            image: nginx:latest  # The container image to use
    ```

    - Apply the file to the cluster and check the result

    ```bash
    kubectl apply -f nginx-deployment.yaml
    kubectl get deployments
    ```

1. Expose the nginx Deployment as a service:

    - Create a file `nginx-deployment-service.yaml` and declare the service in it

    > `nginx-deployment-service.yaml`
    
    ```yaml
    apiVersion: v1                     # The version of the Kubernetes API for services
    kind: Service                      # The kind of Kubernetes object; a Service
    metadata:                          # Metadata about the object
      name: nginx-deployment-service   # The name of the Service
    spec:                              # Specification of the desired behavior of this Service
      selector:                        # Defines how the Service finds which Pods to route traffic to
        app: nginx                     # Selects any Pod with a label having key "app" and value "nginx"
      ports:                           # The list of ports on which the Service listens and forwards traffic
      - protocol: TCP                  # The network protocol this port listens to
        port: 80                       # The port on which the Service is exposed within the cluster
        targetPort: 80                 # The port on the Pod to which the traffic is directed
      type: NodePort                   # Specifies the type of Service; exposes the Service on each node’s IP at a static port
    ```

    - Apply the file to the cluster and check the result

    ```bash
    kubectl apply -f nginx-deployment-service.yaml
    kubectl get svc nginx-deployment-service
    ```

1. Verify in the browser that it works or with `curl localhost:<PORT>`. You get the port number from the `kubectl get svc nginx-deployment-service` command above.

1. Clean up the resources:

    ```bash
    kubectl delete -f nginx-deployment.yaml
    kubectl delete -f nginx-deployment-service.yaml
    ```
    or 

    ```bash
    kubectl delete deployment my-nginx-deployment
    kubectl delete svc nginx-deployment-service
    ```
    or (clean up everything)

    ```bash
    kubectl delete all --all --namespace=default
    ```

> ### Pods vs Deployments

> While you can use Pods directly, in most real-world use cases, especially in production environments, Deployments (or other higher-level constructs like StatefulSets, DaemonSets, etc.) are preferred because they offer management features that standalone Pods do not. Pods are the basic deployable units in Kubernetes, but Deployments provide a higher-level interface that encapsulates the desired state of your application, handling the details of Pod lifecycle, scaling, updates, and more.
