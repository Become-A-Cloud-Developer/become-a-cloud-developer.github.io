+++
title = "Deploy containerized apps on Amazon ECS using CodePipeline"
weight = 5
date = 2024-05-20
draft = false
+++


# Demo - ECS

> Deploy containerized apps on Amazon ECS using CodePipeline

## Prerequisites

### Setting Up a Docker Environment

To begin, you'll need to have the code for a basic web page and a Dockerfile that describes how it will be containerized. Follow these steps if you don´t have one already:

1. **Clone the Repository**: Start by cloning the GitHub repository containing the necessary files.

   ```bash
   git clone https://github.com/larsappel/ECSDemo.git
   ```
   
2. Navigate to the Cloned Directory. Change your current directory to the cloned repository.
	
	```bash
	cd ECSDemo
	```

3. Clean Up. Remove the existing .git directory to start fresh with your setup.

	```bash
	rm -Rf .git
	```

> Note: These commands assume you have Git and Docker already installed on your system. If not, please install them before proceeding.




## Setting Up an AWS CodeCommit Repository

1. Establish a local git code repository to manage your files.

	```bash
	# Initialize a new git repository
	git init
	
	# Add all current files to the repository
	git add .
	
	# Commit the added files with a message
	git commit -m 'Add simple web site'

	```
	
2. Create an AWS CodeCommit code repository and set it as *remote origin* to you local repo

	```bash
	# Create a new CodeCommit repository with a name and description
	aws codecommit create-repository --repository-name DockerDemo --repository-description "Docker Demo Repository"
	```
	
	> Output
	>
	```json
	{
	    "repositoryMetadata": {
	        "accountId": "880731366811",
			...
	        "cloneUrlHttp": "https://git-codecommit.eu-west-1.amazonaws.com/v1/repos/DockerDemo",
			...
	        "Arn": "arn:aws:codecommit:eu-west-1:880731366811:DockerDemo"
	    }
	}
	```
	
	```bash
	# Set CodeCommit repo as origin. Get info from the output above
	git remote add origin https://git-codecommit.eu-west-1.amazonaws.com/v1/repos/DockerDemo
	
	# Push to CodeCommit
	git push -u origin main
	```


## Establishing an Elastic Container Registry (ECR)

1. Create an ECR repository to store your Docker images.

	```bash
	aws ecr create-repository --repository-name dockerdemo
	```

2. Add a `buildspec.yml` file to the _root_ of your code repo

	```yaml
	version: 0.2
	
	phases:
	  pre_build:
	    commands:
	      # Fill in ECR information
	      - REGISTRY_URI=<registry uri>
	      - IMAGE_NAME=<image name>
	      - REGION=<region>
	      # Fill in ECS information
	      - CONTAINER_NAME=MyContainerName # TaskDefinition: container definition name (Wrapper for imageUri)
	      # -----------------------
	      - IMAGE=$REGISTRY_URI/$IMAGE_NAME
	      - COMMIT=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-8)
	      - aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $REGISTRY_URI
	  build:
	    commands:
	      - docker build --tag $IMAGE .
	      - docker tag $IMAGE $IMAGE:$COMMIT
	  post_build:
	    commands:
	      - docker push $IMAGE
	      - docker push $IMAGE:$COMMIT
	      # Create imagedefinitions.json. This is used by ECS to know which docker image to use.
	      - printf '[{"name":"%s","imageUri":"%s"}]' $CONTAINER_NAME $IMAGE:$COMMIT > imagedefinitions.json
	artifacts:
	  files:
	    # Put imagedefinitions.json in the artifact zip file
	    - imagedefinitions.json
	```
	
	> Note: Replace `<registry uri>`, `<image name>`, and `<region>` with your specific ECR details. `CONTAINER_NAME` should match your container definition in the ECS task definition.
	
  
## Setting Up a CodeBuild Project

1. Create a new project in AWS CodeBuild with the following settings:

	- Click on **Create build project**
	- Project name: **BuildDockerDemo**
	- Source
		- Source provider: **AWS CodeCommit**
		- Repository: DockerDemo
		- Branch: main
	- Environment
		- Managed image
		- EC2
		- Operating system: **Ubuntu**
		- Runtime: **Standard**
		- Image: **aws/codebuild/standard:7.0**
		- Privileged: **Yes** (check)
		- New service role
		- Role name: codebuild-BuildDockerDemo-service-role
		- Use a buildspec file
	- Click on **Create build project**


2. After creating the project, set up permissions that will grant it access to write to the ECR container registry service

	- Find the newly created role: *codebuild-BuildDockerDemo-service-role*
	- Add the managed policy: **AmazonEC2ContainerRegistryPowerUser**

1. Click: Start build

2. Verify that the image is in the ECR repository

> Note: The settings provided above are specific to this setup. Adjust them according to your project name.

## Establishing an ECS Cluster

1. Create a Task Definition: 
	- Task definition family: **DockerDemoTask**
	- Use AWS Fargate
	- Container definition name: **DockerDemoContainer**
	- Image URI: Paste the URI of the docker image
	- Click on **Create**

1. Create an ECS Cluster:
	- Cluster name: **DockerDemoCluster**
	- Click on **Create**

1. Create a Service within the cluster:
	- Navigate to the Services tab in your cluster and click on Create
	- Family: **DockerDemoTask**
	- Service name: **DockerDemoService**
	- Replica: 1
	- Expand the Networking section and **Choose or create a security group open for HTTP**
	- Click on **Create**

4. Verify that the service started a task/container. Find the public IP and open in a browser

> Note: Ensure all names and URIs are correctly entered. The settings provided here are specific to this demonstration.


## Setting Up a CodePipeline

> **Important!**
> 
> Before creating the pipeline, update the `buildspec.yml` file with the *Task Definition Container Name* and commit the changes. Don´t forget to push the change to the AWS CodeCommit code repo

1. Create a pipeline
	- Pipeline name: **DockerDemoPipeline**
	- Accept the new service role (AWSCodePipelineServiceRole-eu-west-1-DockerDemoPipeline)
	- Click on **Next**
	- Source provider: **AWS CodeCommit**
	- Repository name: DockerDemo
	- Branch name: main
	- Trigger Method: Amazon CloudWatch Events (recommended)
	- **Artifact Store:** CodePipeline default
	- Click on **Next**
	- Build provider: **AWS CodeBuild**
	- Project name: BuildDockerDemo
	- Click on **Next**
	- Deploy provider: **Amazon ECS**
	- Cluster name: **DockerDemoCluster**
	- Service name: **DockerDemoService**
	- Click on **Next**
	- Click on **Create pipeline**

> Note: Ensure all settings, names, and URIs are correctly configured. Update the buildspec.yml file to reflect your setup.

2. Verify that the pipeline works
	- Browse to the service
	- Change the code and commit. See how the change trigger the pipeline
	- See the cluster service task

## Configuring a Load Balanced Scalable Cluster Service

1. Create an additional cluster **service** with load balancing capabilities:
	- Replica Count: **2** (for scalability)
	- Load balancer type: **Application Load Balancer**
	- Load balancer name: **DockerDemoALB**
	- Target group name: **DockerDemoTG**
	- Click on **Create**


2. Verify by going to the A**pplication Load Balancer (ALB) DNS** and ensure it is operating correctly.



# Short Description of All AWS Service Used


- **AWS CodeBuild**: A fully managed continuous integration service that compiles source code, runs tests, and produces software packages that are ready to deploy.

- **AWS CodeCommit**: A hosted source control service that you can use to privately store and manage code in the cloud.

- **AWS Fargate**: A serverless compute engine for containers that works with both Amazon Elastic Container Service (ECS) and Amazon Elastic Kubernetes Service (EKS), eliminating the need to manage servers.

- **AWS CodePipeline**: A continuous integration and continuous delivery (CI/CD) service that automates the steps required to release your software changes.

- **Amazon ECS (Elastic Container Service)**: A highly scalable, high-performance container management service that supports Docker containers and allows you to run applications on a managed cluster.

- **AWS IAM (Identity and Access Management)**: A web service that helps you securely control access to AWS resources.

- **AWS Application Load Balancer (ALB)**: A service that automatically distributes incoming application traffic across multiple targets, such as Amazon EC2 instances, containers, and IP addresses.


The `imagedefinitions.json` file is an important component in AWS ECS (Elastic Container Service), particularly when used in conjunction with AWS CodePipeline and AWS CodeBuild for continuous integration and continuous deployment (CI/CD) workflows. Here's a detailed explanation of its function and usage:

# Purpose of the File `imagedefinitions.json`

The `imagedefinitions.json` file is a critical link between the build process and the deployment process in a CI/CD pipeline, when using AWS services. It ensures that ECS can automatically deploy the correct Docker image based on the most recent build.
 
The file specifies the Docker container images to be used by the ECS service when you deploy a new version of your application. It describes the **Container Image Specification**.

The file plays a crucial role in the integration between AWS CodePipeline, AWS CodeBuild, and Amazon ECS. During the CI/CD process, a new Docker image is built and pushed to a registry (like AWS ECR - Elastic Container Registry), and `imagedefinitions.json` is updated with the new image details.

An important aspect is **Version Control**, where the file ensures that the exact version of the image that has passed through the build and test stages is the one deployed.

### Format and Content

The `imagedefinitions.json` file typically contains JSON-formatted data that specifies the container name and image URI. Here’s a basic example:

```json
[
  {
    "name": "container-name",
    "imageUri": "registry/repository:tag"
  }
]
```

- **`name`**: This is the name of the container as defined in the ECS task definition. It must match the name specified in the task definition.

- **`imageUri`**: This is the URI of the Docker image. It usually includes the registry path, repository name, and the image tag (which often represents different versions of the image).

### Workflow Integration

1. **Build Stage**: In the build stage of your CI/CD pipeline (handled by AWS CodeBuild), your application is built, and a new Docker image is created and pushed to a repository (like ECR).

2. **Update `imagedefinitions.json`**: After pushing the image, the build process updates the `imagedefinitions.json` file with the new image URI.

3. **Artifact for Deployment**: This file is then passed as an artifact to the deployment stage.

4. **Deployment Stage**: During the deployment stage, AWS CodePipeline triggers ECS to deploy the new version of the application. ECS references the `imagedefinitions.json` file to determine which image to deploy.

5. **ECS Update**: ECS updates the service with the new task definition that points to the Docker image specified in the `imagedefinitions.json`, thus rolling out the new version of the application.


# Delete Resources

Delete:
1. Task
2. Cluster
3. Namespace
4. ALB (CF)
5. TG (CF)
6. ECR
7. codebuild-BuildDockerDemo-service-role
8. CodeBuildBasePolicy-codebuild-BuildDockerDemo-service-role-eu-west-1
9. Build Project
10. CodeCommit repo