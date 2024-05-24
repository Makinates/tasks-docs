# How to Deploy to ECS from the AWS Console

Amazon Elastic Container Service (ECS) is a fully-managed container orchestration service provided by Amazon Web Services (AWS). It allows you to deploy, manage, and scale containerized applications across a cluster of Amazon Elastic Compute Cloud (EC2) instances.

## Prerequisites

Before you begin, ensure that you have the following:

- An AWS account. If you don't have one, you can sign up for a free tier account at <https://aws.amazon.com>.

- A Docker image that you want to deploy to ECS. This image should be available in a Docker registry, such as Docker Hub or Amazon Elastic Container Registry (ECR). This walk-through will you the `simple-node-app` image from a [previous step](./02%20-%20How%20to%20Push%20a%20Container%20to%20Docker%20Hub.md)

## Step 1: Create an ECS Cluster

An ECS cluster is a logical grouping of EC2 instances that run your containerized applications.

- Open the AWS Management Console and navigate to the ECS service.

- Click on "Create Cluster" and choose the desired cluster configuration options.

- Click "Create" to create the ECS cluster.

![Create ECS Cluster](./images/ecs-console/ecs1.png)

![Creating ECS Cluster Config](./images/ecs-console/ecs2.png)

## Step 1.5: Create ECS Task Execution Role

ECS will need permissions and the ability to execute tasks and spin up/down services on your behalf. To enable ECS to this, we will create a role with the predefined ECS Task Execution permissions.

- Navigate to AWS IAM Dashboard to create the role with the neccessary permissions.

![IAM Role](./images/ecs-console/ecs3.5.png)

![IAM Role](./images/ecs-console/ecs3.6.png)

![IAM Role](./images/ecs-console/ecs3.7.png)

![IAM Role](./images/ecs-console/ecs3.8.png)

![IAM Role](./images/ecs-console/ecs3.9.png)

## Step 2: Create an ECS Task Definition

A task definition is a JSON file that describes one or more containerized applications, including the Docker image, resource requirements, and networking settings.

- In the ECS service, navigate to the "Task Definitions" section.

![Task Definition](./images/ecs-console/ecs3.png)

- Click "Create new Task Definition" and choose the desired configuration options.

![Create New Task Definition](./images/ecs-console/ecs4.png)

- In the "Container Definitions" section, provide the details of your Docker image, such as the image URI, CPU and memory requirements, and port mappings.

![Defining Task Definition Config](./images/ecs-console/ecs5.png)

![Defining Task Definition Container](./images/ecs-console/ecs6.png)

- Click "Create" to create the ECS task definition.

## Step 3: Create an ECS Service

An ECS service is responsible for running and maintaining the desired number of tasks based on the task definition.

- In the ECS service, navigate to the "Clusters" section and select your ECS cluster.

- Click "Create" under the "Services" tab.

![Create Service](./images/ecs-console/ecs7.png)

- Configure the service settings, such as the task definition, desired number of tasks, and networking options.

![Service Config 1](./images/ecs-console/ecs8.png)

![Service Config 2](./images/ecs-console/ecs9.png)

![Service Config 3](./images/ecs-console/ecs10.png)

![Service Config 4](./images/ecs-console/ecs11.png)

- Click "Next" and follow the prompts to complete the service creation process.

![Service Creation Success](./images/ecs-console/ecs12.png)

## Step 4: Verify the Deployed Application

After successfully creating the ECS service, your containerized application should be deployed and running on the ECS cluster.

- In the ECS service, navigate to the "Clusters" section and select your ECS cluster.
- Under the "Services" tab, you should see your deployed service with the desired number of running tasks. Navigate to the `Configuration and networking` tab to get the Load Balancer's DNS Address

![Getting the LB Address](./images/ecs-console/ecs12.5.png)

- Visit the address in your browser to view your application.

![View App in Browser](./images/ecs-console/ecs13-1.png)

- Refresh the browser a few times to see the load balancer distributing traffic across the number of running tasks and containers.

![Refresh to see LB at work](./images/ecs-console/ecs13-2.png)

Congratulations! You have successfully deployed your Docker image to Amazon Elastic Container Service (ECS) in the AWS Management Console with and Application Load Balancer to distribute the traffic.
