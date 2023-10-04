## High Availability Web Server

### Backround
High availability is crucial in any cloud because it ensures applications and services are
always available to customer, even in the face of failures or disruptions. Cloud
environments are designed to be highly scalable, but even the most reliable systems can
experience downtime due to hardware failures, network outages, or other unexpected
issues.
In a traditional on-premises environment, ensuring high availability typically involves
duplicating hardware and configuring failover mechanisms to ensure that if one server or
component fails, another one can take over. 

However, this approach can be costly and
complex to implement and maintain.
Cloud environments provide a more cost-effective and scalable solution for achieving
high availability. By leveraging the cloud's global infrastructure, you can distribute your
application across multiple availability zones (AZs) to ensure that it remains available
even if one AZ experiences a failure. Additionally, you can use auto-scaling groups to
automatically add or remove instances as demand fluctuates.
Overall, high availability is critical in the cloud to ensure that your applications and
services are always accessible to your users, regardless of any unexpected issues that
may arise.

By implementing a highly available web server in AWS, you can ensure that
your users have a seamless experience, even in the face of disruptions or failures.
To achieve web server high availability in the cloud, Autoscaling is one of the main
component allows you to automatically adjust the capacity of your AWS resources based
on demand. With Auto Scaling, you can ensure that you have enough resources to handle
sudden spikes in traffic or usage, while also minimising costs during periods of low
demand.
Auto Scaling works by monitoring your application's performance metrics, such as CPU
utilisation or network traffic, and then adjusting the number of instances or resources
accordingly. For example, if your application experiences a sudden surge in traffic, Auto
Scaling can automatically launch new instances to handle the increased load. Similarly, if
traffic decreases, Auto Scaling can automatically terminate instances to reduce costs.
Auto Scaling can be used in conjunction with other AWS services, such as Elastic Load
Balancing and Amazon EC2, to create highly available and scalable applications. By
leveraging Auto Scaling, you can ensure that your applications are always available to
your users, while also optimising costs and minimising manual intervention.

In this demo i will be implementing a full automated pipeline to deploy a highly available
web server within AWS using Github Action, Packer, and Terraform.
The objective of this lab is to demonstrate how to use modern tools to automate the
process of deploying a web server in AWS, ensuring high availability and scalability.
To achieve this, we will use Github Action, Packer, and Terraform, which are powerful
tools that enable automation of software deployment, infrastructure provisioning, and
configuration management.

### Repositories and Configuration File

Suppose i am part of a team that is responsible for managing the web server and its
infrastructure. I decided to use Github private repositories to manage my cloud
configuration file where i can version control my infrastructure and keep track of all
changes made over time. This allows me to maintain an audit trail of my infrastructure,
reproduce the exact same environment in different regions, create consistency across my
infrastructure and collaborate with my team as it provides a single source of truth for
reviewers. On the other hand, manual configuration via a GUI or click ops can lead to
differences between environments, making it difficult to reproduce issues.
The following is the repository structure:

.

├── .gitignore

├── .github/workflows

├── README.md

├── arch

├── golden_image

└── infrastructure

.gitignore is a file used by Git to specify files and directories that should be ignored
when tracking changes in a project. These files typically include build artifacts, temporary
files, and other files that are not essential to the project's source code. By using
.gitignore, you can prevent these files from being committed to the repository, which
helps keep the repository clean and organized.
.github/workflows folder is a directory in a GitHub repository that contains YAML files
defining Continuous Integration/Continuous Deployment (CI/CD) workflows. These
workflows are automated scripts that run in response to events in a repository, such as a
pull request being opened or code being pushed to a branch.

README.md is a file in a GitHub repository that contains a brief description of the project,
its purpose, and any other relevant information for potential contributors or users. The
README file is typically the first file that someone will see when they visit a repository on
GitHub, and it is used to provide a high-level overview of the project.
arch is the folder where all diagram exists
golden_image is the folder that contain packer configuration to build the web server
infrastructure is the folder that contains Terraform Cloud configuration such
networking, security, Autoscaling etc...

### Infrastructure as Code using Terraform

Infrastructure as Code (IaC) is the practice of defining and managing Cloud infrastructure
using code. With IaC, we can use version control systems like Github repositories to track
and manage infrastructure changes over time, enabling automation of deployment and
management of infrastructure.
Terraform is an open-source infrastructure as code tool that allows describing
infrastructure in a high-level configuration language and then automatically deploy it
across multiple cloud providers, such as AWS, Azure, and Google Cloud. With Terraform,
we can create, modify, and destroy resources as needed, and the tool will manage the
dependencies and order of operations required to ensure a successful deployment.
Terraform allows to define infrastructure as code in a declarative syntax, which is more
intuitive and easier to maintain than imperative scripts. We can define resources and their
dependencies, set variables, and configure inputs and outputs in your configuration files.
One of the key benefits of using Terraform is its ability to provide consistency and
repeatability in our infrastructure deployments. Since Terraform is code-based, we can
easily reproduce your infrastructure in different environments, such as development,
staging, and production. This reduces the risk of manual errors and streamlines the
process of managing and scaling your infrastructure.
As mentioned before, Terraform configuration file resides in the infrastructure folder
which contains the following files:

### 1- AppLoadBalancer.tf Application Load Balancer

Application Load Balancer (ALB) are designed to distribute incoming traffic across
multiple targets, such as Amazon Elastic Compute Cloud (EC2) instances, containers, or
IP addresses. ALB operates at the application layer (Layer 7) of the OSI model. This
means that it can route traffic based on the content of HTTP requests, such as the URL
path, host header, or query string. The AWS ALB provides several advanced features,
such as content-based routing, path-based routing, and host-based routing. It also
supports SSL/TLS encryption, sticky sessions, and health checks to ensure that only
healthy targets receive traffic. One of the key benefits of using the AWS ALB is its ability
to improve the availability and scalability of your application. The ALB can automatically
distribute incoming traffic to healthy targets, while also providing fault tolerance and
redundancy in case of failures.

ALB Terraform Configuration are as the following:

```hcl
resource "aws_lb" "alb" {
name = "web-alb"
internal = false
load_balancer_type = "application"
security_groups = [aws_security_group.sg_alb.id]
subnets = aws_subnet.alb_subnet.*.id
enable_cross_zone_load_balancing = true
enable_deletion_protection = var.alb_termination
tags = {

Name = "public-
lb${var.client_name}-${var.environment}-${data.aws_region.current.name}"

}
}
// ALB Listener
resource "aws_lb_listener" "alb_listener" {
load_balancer_arn = aws_lb.alb.arn
port = var.listener_port
protocol = var.listener_protocol
default_action {
type = "forward"
target_group_arn = aws_lb_target_group.asg_target_group.arn
}
}
#################################################
# Target Group
#################################################
resource "aws_lb_target_group" "asg_target_group" {
target_type = "instance"
name = "tg-webservers"

port = var.tg_port
protocol = var.tg_protocol
vpc_id = aws_vpc.vpc.id
load_balancing_algorithm_type = "round_robin"
health_check {
enabled = true
interval = var.health_check_interval
path = var.health_check_path
matcher = var.health_check_matcher
timeout = var.health_check_timeout
port = var.health_check_port
protocol = var.health_check_protocol
}
tags = {

Name = "tg-
webservers${var.client_name}-${var.environment}-${data.aws_region.current.name

}
}
```

https://github.com/AdamHmouda/High-Availability-Web-Server
