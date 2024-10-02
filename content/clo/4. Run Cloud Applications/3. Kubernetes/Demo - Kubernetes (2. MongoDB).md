+++
title = 'Kubernetes MongoDB'
weight = 2
date = 2024-10-02
draft = false
+++

# Demo - Kubernetes MongoDB

### Persistent Storage

In Docker Desktop's Kubernetes, a dynamic `StorageClass` is provided out-of-the-box. This means you can create PersistentVolumeClaims (PVCs), and the storage will be provisioned dynamically.

### Deploy MongoDB using StatefulSet:

1. Create a **StatefulSet** for the MongoDB. Create the file `mongodb-statefulset.yaml`

	> `mongodb-statefulset.yaml`

	```yaml
	apiVersion: apps/v1
	kind: StatefulSet
	metadata:
	  name: mongodb
	spec:
	  selector:
	    matchLabels:
	      app: mongodb
	  replicas: 1
	  serviceName: mongodb-service
	  template:
	    metadata:
	      labels:
	        app: mongodb
	    spec:
	      containers:
	      - name: mongodb
	        image: mongo:latest
	        ports:
	        - containerPort: 27017
	        volumeMounts:
	        - name: mongodb-data
	          mountPath: /data/db
	  volumeClaimTemplates:
	  - metadata:
	      name: mongodb-data
	    spec:
	      accessModes:
	        - ReadWriteOnce
	      resources:
	        requests:
	          storage: 1Gi
	```

	- Apply the file to the cluster and check the result

	```bash
	kubectl apply -f mongodb-statefulset.yaml
	```

1. Create a **Service** for the MongoDB. Create the file `mongodb-service.yaml`. This service will expose MongoDB within the cluster:

	> `mongodb-service.yaml`

	```yaml
	apiVersion: v1
	kind: Service
	metadata:
	  name: mongodb-service
	spec:
	  selector:
	    app: mongodb
	  ports:
	    - protocol: TCP
	      port: 27017
	      targetPort: 27017
	```

	- Apply the file to the cluster and check the result

	```bash
	kubectl apply -f mongodb-service.yaml
	```

### Deploy mongo-express:

1. Create a **Deployment** for the Mongo Express. Create the file `mongodb-service.yaml`.

	> `mongo-express-deployment.yaml`

	```yaml
	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: mongo-express
	spec:
	  replicas: 1
	  selector:
	    matchLabels:
	      app: mongo-express
	  template:
	    metadata:
	      labels:
	        app: mongo-express
	    spec:
	      containers:
	      - name: mongo-express
	        image: mongo-express:latest
	        ports:
	        - containerPort: 8081
	        env:
	        - name: ME_CONFIG_MONGODB_SERVER
	          value: "mongodb-service"
	```

	- Apply the file to the cluster and check the result

	```bash
	kubectl apply -f mongo-express-deployment.yaml
	```

1. Create a **Service** for the Mongo Express UI. Create the file `mongo-express-service.yaml`. 

	> `mongo-express-service.yaml`

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
	  type: NodePort
	```

	- Apply the file to the cluster and check the result

	```bash
	kubectl apply -f mongo-express-service.yaml
	```

1. Verify by running

	```bash
	kubectl get all
	kubectl get pvc
	```
	Open a browser and go to `service/mongo-express-service`


> ### NodePort
> With `type: NodePort`, mongo-express will be exposed on a port in the range of 30000-32767 on your Docker Desktop's Kubernetes node (which is essentially your machine). You can use `kubectl get svc` to see on which port it has been exposed.


### Initialize MongoDB

The following steps will initialize the MongoDB with a database, a collection and some `todo-items` using a Kubernetes Job

1. Create a script to initialize MongoDB in a file called `init-mongo.sh`.

   > `init-mongo.sh`

   ```bash
	#!/bin/bash
	set -e
	
	# Wait for MongoDB to be ready
	for i in {1..30}; do
	    if mongosh --host $MONGO_HOST --port $MONGO_PORT --eval "db.stats()" > /dev/null; then
	        break
	    fi
	    echo "Waiting for MongoDB to start..."
	    sleep 2
	done
	
	mongosh --host $MONGO_HOST --port $MONGO_PORT <<EOF
	use ToDoAppDb
	db.ToDoItems.insert({ Title: 'Todo1', IsCompleted: false, Details: 'Initialize MongoDB' })
	db.ToDoItems.insert({ Title: 'Todo2', IsCompleted: true, Details: 'Run Kubernetes Job' })
	EOF
   ```

   - Ensure the script is executable:

   ```bash
   chmod +x init-mongo.sh
   ```

2. Package the script in a Docker image.

   > `Dockerfile`

   ```Dockerfile
   FROM mongo:latest
   COPY init-mongo.sh /init-mongo.sh
   CMD [ "/init-mongo.sh" ]
   ```

   - Build the Docker image and push it to Docker Hub:
   
   ```bash
   docker build -t larsappel/my-mongo-init:latest .
   docker push larsappel/my-mongo-init:latest
   ```

3. Use a Kubernetes **Job** to run the initialization.

   > `mongo-init-job.yaml`

   ```yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: mongo-init-job
   spec:
     template:
       spec:
         containers:
         - name: mongo-init-container
           image: larsappel/my-mongo-init:latest
           env:
           - name: MONGO_HOST
             value: mongodb-service
           - name: MONGO_PORT
             value: "27017"
           # If MongoDB requires authentication, add the environment variables here.
         restartPolicy: OnFailure
   ```

   - Apply the job to run the initialization and verify in the logs:

   ```bash
   kubectl apply -f mongo-init-job.yaml
   ```

   - Once this job completes, your MongoDB instance should be initialized with a `ToDoAppDb` database containing a `ToDoItems` collection with two items. Verify in mongo-express or in the logs:

   ```bash
   kubectl logs --selector job-name=mongo-init-job
   ```

> ###Job
> Executes one-off tasks, such as batch processing, one-time setups, or any other short-lived tasks.

### Deploy an ASP.NET Core WebApp

The following steps will deploy an ASP.NET Core Razor Pages web app in the Kubernetes cluster

1. Create a **ConfigMap** in a file called `todo-configmap.yaml` file for the MongoDB settings. The data will be inserted as environment variables in the todo app deployment below

   > `todo-configmap.yaml`

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: app-config
   data:
     MongoDbSettings__ConnectionString: "mongodb://mongodb-service:27017"
     MongoDbSettings__DatabaseName: "ToDoAppDb"
     MongoDbSettings__CollectionName: "ToDoItems"
     TODO_SERVICE_IMPLEMENTATION: MongoDb
   ```

   - Apply the ConfigMap:

   ```bash
   kubectl apply -f todo-configmap.yaml
   ```


1. Specify a Kubernetes Deployment in a new file called `todo-deployment.yaml` for the `larsappel/todo` app.

   > `todo-deployment.yaml`

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: todo-webapp
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: todo-webapp
     template:
       metadata:
         labels:
           app: todo-webapp
       spec:
         containers:
         - name: todo-container
           image: larsappel/todo:latest
           ports:
           - containerPort: 80
           envFrom: 
           - configMapRef:
               name: app-config
   ```

   - Apply the deployment configuration:

   ```bash
   kubectl apply -f todo-deployment.yaml
   ```

2. Expose the web app using a service. Create a `todo-service.yaml` file.

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
     type: NodePort
   ```

   - Apply the service configuration:

   ```bash
   kubectl apply -f todo-service.yaml
   ```
1. Verify

   ```bash
   kubectl get all
   ```
	Open a browser and go to `service/todo-service`
