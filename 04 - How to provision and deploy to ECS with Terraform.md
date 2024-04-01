# How to provision and deploy to ECS with Terraform

Amazon Elastic Container Service (ECS) is a fully-managed container orchestration service provided by Amazon Web Services (AWS). It allows you to deploy, manage, and scale containerized applications across a cluster of Amazon Elastic Compute Cloud (EC2) instances.

Terraform is an open-source Infrastructure as Code (IaC) tool that allows you to provision and manage cloud resources, including Amazon Elastic Container Service (ECS), in a declarative way. In this documentation, we'll guide you through the process of deploying a Docker image to ECS using Terraform.

## Prerequisites

Before you begin, ensure that you have the following:

1. An AWS account. If you don't have one, you can sign up for a free tier account at [https://aws.amazon.com](https://aws.amazon.com/).

2. Terraform installed on your machine. You can download Terraform from the official website: [https://www.terraform.io/downloads.html](https://www.terraform.io/downloads.html). You can also use [tfenv](https://github.com/tfutils/tfenv) to install and manage Terraform versions.

3. The AWS Command Line Interface (CLI) installed and configured on your machine. You can find instructions for installing and configuring the AWS CLI at [https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html).

4. A Docker image that you want to deploy to ECS. This image should be available in a Docker registry, such as Docker Hub or Amazon Elastic Container Registry (ECR). This walk-through will you the `simple-node-app` image from a [previous step](./02%20-%20How%20to%20Push%20a%20Container%20to%20Docker%20Hub.md)

In this guide, we will write Terraform code to create the following resources:

- An ECS cluster
- An IAM role and policy attachment for the ECS task execution role
- An ECS task definition for our `simple-node-app` Docker image
- An ECS service to run the task definition

## Step 1: Configure Terraform Resources

> Terraform allows you define all resources in one file or in multiple files. For better code maintenance and easy understanding, we will break the code into separate files.

- Create a new file named `main.tf`. First, we will define a block that creates an Amazon Elastic Container Service (ECS) cluster with the name `makinode-app-cluster`. An ECS cluster is a logical grouping of EC2 instances or Fargate tasks that run your containerized applications.

```hcl
# Create the ECS cluster
resource "aws_ecs_cluster" "makinode_app_cluster" {
  name = "makinode-app-cluster"
}
```

- Next, we will create an AWS Identity and Access Management (IAM) role named "ecs-task-execution-role". This role is used by the ECS tasks to access other AWS resources, such as ECR (Elastic Container Registry) or CloudWatch Logs. The `assume_role_policy` defines the permissions for the ECS tasks to assume this role.

```hcl
resource "aws_iam_role" "ecs_task_role" {
  name = "ecs-task-execution-role"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
}
```

- Attach the `AmazonECSTaskExecutionRolePolicy` managed policy to the `ecs_task_role` created in the previous block. This policy provides the necessary permissions for the ECS tasks to run and interact with other AWS services.

```hcl
resource "aws_iam_role_policy_attachment" "ecs_task_role_policy_attachment" {
  role       = aws_iam_role.ecs_task_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
```

- Create an ECS task definition named `makinode-app-task`. A task definition is a JSON file that describes one or more containerized applications, including the Docker image, resource requirements, and networking settings.

```hcl
# Create the task definition
resource "aws_ecs_task_definition" "makinode_app_task" {
  family                   = "makinode-app-task"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"

  execution_role_arn = aws_iam_role.ecs_task_role.arn

  container_definitions = <<DEFINITION
    [
        {
            "name": "makinode-app-container",
            "image": "${var.container_image_url}",
            "portMappings": [
                {
                    "containerPort": 3001,
                    "hostPort": 3001
                }
            ]
        }
    ]
    DEFINITION
}
```

> `family`: Specifies the family name for the task definition.
>
> `network_mode`: Specifies the networking mode for the task, in this case, "awsvpc" (AWS Virtual Private Cloud).
>
> `requires_compatibilities`: Specifies that the task definition is compatible with the Fargate launch type.
>
> `cpu` and `memory`: Specify the CPU and memory resources to be allocated for the task.
>
> `execution_role_arn`: Specifies the IAM role (`ecs_task_role`) that the task should assume.
>
> `container_definitions`: Specifies the container definition for the Docker image. It includes the name of the container, the Docker image URL (here, we use a variable defined in `variables.tf`), and the port mappings for the container.

- Create an ECS service named `makinode-app-service`. An ECS service is responsible for running and maintaining the desired number of tasks based on the task definition.

```hcl
# Create the ECS service
resource "aws_ecs_service" "makinode_app_service" {
  name            = "makinode-app-service"
  cluster         = aws_ecs_cluster.makinode_app_cluster.id
  task_definition = aws_ecs_task_definition.makinode_app_task.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = [aws_subnet.public-subnet[0].id, aws_subnet.public-subnet[1].id]
    assign_public_ip = true
    security_groups  = [aws_security_group.makinode_app_sg.id]
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.makinode_app_tg.arn
    container_name   = "makinode-app-container"
    container_port   = 3001
  }

  depends_on = [aws_lb_listener.makinode_app_listener]
}
```

> `name`: Specifies the name of the ECS service.
>
> `cluster`: Specifies the ECS cluster where the service should run, using the ID of the `makinode_app_cluster` cluster created earlier.
>
> `task_definition`: Specifies the task definition (`makinode_app_task`) to be used for running the tasks.
>
> `desired_count`: Specifies the desired number of tasks to be running in the service, in this case, 2.
>
> `launch_type`: Specifies the launch type for the tasks, in this case, "FARGATE".
>
> `network_configuration`: Specifies the networking configuration for the tasks, including the subnets, public IP assignment, and security groups. In this case we reference subnets defined in our `networking.tf` file, and the security group defined in our `security.tf` file.
>
> `load_balancer`: Configures the load balancer settings for the service, including the target group ARN (`makinode_app_tg`), container name, and container port. The load balancer is defined in the `loadbalancer.tf` file.
>
> `depends_on`: Specifies that the ECS service depends on the `makinode_app_listener` resource, which is not shown in the provided code.

This Terraform code sets up an ECS cluster, task definition, and service to deploy and run your Docker image on the AWS Fargate launch type. It also configures the necessary IAM role and network settings for the tasks to run and interact with other AWS services.
