# AWS Notes

## Table of Contents
<!-- MDTOC maxdepth:2 firsth1:2 numbering:0 flatten:0 bullets:1 updateOnSave:1 -->

- [Table of Contents](#Table-of-Contents)   
- [AWS -- Amazon Web Services](#AWS-Amazon-Web-Services)   
- [IAM -- Identity and Access Management](#IAM-Identity-and-Access-Management)   
   - [Roles](#Roles)   
   - [Security Group](#Security-Group)   
   - [Elastic Load Balancer](#Elastic-Load-Balancer)   
- [EC2 -- Elastic Compute Cloud](#EC2-Elastic-Compute-Cloud)   
- [S3 -- Simple Storage Service](#S3-Simple-Storage-Service)   
- [ELB -- Elastic Load Balancing](#ELB-Elastic-Load-Balancing)   
- [Route 53 -- DNS Servicwe](#Route-53-DNS-Servicwe)   
- [RDS -- Relational Database Service](#RDS-Relational-Database-Service)   
- [ElastiCache](#ElastiCache)   
- [ECR -- EC2 Container Registry](#ECR-EC2-Container-Registry)   
- [ECS -- EC2 Container Service](#ECS-EC2-Container-Service)   
   - [ECS CLI](#ECS-CLI)   
   - [Clusters](#Clusters)   
   - [Prepare the S3 bucket for ECS Container Agent configuration](#Prepare-the-S3-bucket-for-ECS-Container-Agent-configuration)   
   - [ECS Container Agent](#ECS-Container-Agent)   
   - [Container Instances](#Container-Instances)   
   - [Task Definitions](#Task-Definitions)   
   - [Running Tasks](#Running-Tasks)   
   - [Scheduling Services](#Scheduling-Services)   
   - [Tearing Down a Cluster](#Tearing-Down-a-Cluster)   
   - [Ruby on Rails App example](#Ruby-on-Rails-App-example)   
- [Glossary](#Glossary)   
- [Tools, tips](#Tools-tips)   
   - [Troubleshooting](#Troubleshooting)   

<!-- /MDTOC -->

## AWS -- Amazon Web Services


## IAM -- Identity and Access Management

### Roles

IAM roles are a secure way to grant permissions to entities that you trust on AWS resources. Examples of entities include the following:

 - Application code running on an EC2 instance that needs to perform actions on AWS resources
 - IAM user in another account
 - An AWS service that needs to act on resources in your account to provide its features
 - Users from a corporate directory who use identity federation with SAML

IAM roles issue keys that are valid for short durations, making them a more secure way to grant access.


### Security Group

https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html?icmpid=docs_ec2_console

A security group acts as a virtual firewall that controls the traffic for one or more instances. When you launch an instance, you associate one or more security groups with the instance. You add rules to each security group that allow traffic to or from its associated instances.

You can modify the rules for a security group at any time; the new rules are automatically applied to all instances that are associated with the security group. When we decide whether to allow traffic to reach an instance, we evaluate all the rules from all the security groups that are associated with the instance.

### Elastic Load Balancer

https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html?icmpid=docs_elbv2_console

Elastic Load Balancing distributes incoming application traffic across multiple EC2 instances, in multiple Availability Zones. This increases the fault tolerance of your applications.

The load balancer serves as a single point of contact for clients, which increases the availability of your application. You can add and remove instances from your load balancer as your needs change, without disrupting the overall flow of requests to your application. Elastic Load Balancing scales your load balancer as traffic to your application changes over time, and can scale to the vast majority of workloads automatically.

You can configure health checks, which are used to monitor the health of the registered instances so that the load balancer can send requests only to the healthy instances. You can also offload the work of encryption and decryption to your load balancer so that your instances can focus on their main work.


## EC2 -- Elastic Compute Cloud



## S3 -- Simple Storage Service



## ELB -- Elastic Load Balancing



## Route 53 -- DNS Servicwe



## RDS -- Relational Database Service

Engines supported:

 - MySQL
 - MariaDB
 - Postgres
 - MsSQL
 - Oracle
 - Amazon Aurora (not eligable on free tier)


## ElastiCache

Engines supported:

 - Redis
 - Memcached


 ## ECR -- EC2 Container Registry

 *ECR* is managed Docker Registry.

 It's part of ECS, but in the CLI it is a separate "service".
 
 Example commands:

  - `aws ecr get-login` -- Generates a `docker login` command that logs you intto the AWS Container Registry for 12 hours.
  - `aws ecr create-repository --repository-name deepdive/nginx` -- Create a new repository
  - `aws ecr describe-repositories` -- Describe all repositories
  - `aws ecr list-images --repository-name deepdive/nginx` -- List all images in the deepdive/nginx repository
  - `docker pull nginx:1.9` -- Pull down the nginx image so we can push it later
  - `docker tag nginx:1.9 xxx.dkr.ecr.us-east-1.amazonaws.com/deepdive/nginx:1.9` -- Tag the nginx image
  - `docker push xxx.dkr.ecr.us-east-1.amazonaws.com/deepdive/nginx` -- Push the nginx image to your repository


## ECS -- EC2 Container Service

### ECS CLI

 - Amazon [ECS CLI on GitHub](https://github.com/aws/amazon-ecs-cli)
 - Supports working with `docker-compose` files. Not available on Windows.
 - Amazon's [tutorial on the ECS CLI](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_tutorial.html)

### Clusters

A *Cluster* is a group of container instances that act as a single computing resource.

There is the `default` cluster, if you don't specify a cluster name.

Example commands:
 - `aws ecs create-cluster --cluster-name deepdive` -- Create the 'deepdive' cluster
 - `aws ecs list-clusters`
 - `aws ecs describe-clusters --clusters deepdive`
 - `aws ecs delete-cluster --cluster deepdive`

### Prepare the S3 bucket for ECS Container Agent configuration

Example commands:

 - `aws s3api create-bucket --bucket tkarakai-gto-deepdive` -- Create the S3 bucket for the deepdive cluster
 - `aws s3 cp ecs.config s3://tkarakai-gto-deepdive/ecs.config` -- Copy the ECS config file you downloaded to the S3 bucket
 - `aws s3 ls s3://tkarakai-gto-deepdive` -- Verify the ECS config is in the S3 bucket

### ECS Container Agent

The *Container Agent* is a tool installed on the EC2 Instances that allows the instance to join the ECS Cluster.

 - Written in GoLang
 - Open Source, [available on GitHub](https://github.com/aws/amazon-ecs-agent)
 - There is a [Docker Image](https://hub.docker.com/r/amazon/amazon-ecs-agent/) for it
 - There are [ECS Optimized AMIs](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-optimized_AMI.html) per region that we are free to use. It is Amazon tested, contains a not too old version od Docker installed and the ECS Container Agent (as a Docker Container). Take their AMI ID to be used in the `aws ec2 run-instances --image-id [...] ...` command.
 - Lots of [configurations](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-agent-config.html), most of which are via environment variables.
 - If your container instance was launched **with the Amazon ECS-optimized AMI**, you can set these environment variables in the `/etc/ecs/ecs.config` file and then restart the agent.
 - **If you are manually starting** the Amazon ECS container agent (for non-Amazon ECS-optimized AMIs), you can use these environment variables in the `docker run` command that you use to start the agent with the syntax `--env=VARIABLE_NAME=VARIABLE_VALUE`.

### Container Instances

*Container Instance* is an EC2 Instance which is part of a Cluster, not to be confused with a *Container* that is a Docker Container started via a *Task*.

Example commands:

 - `aws ec2 run-instances --image-id ami-1c002379 --count 3 --instance-type t2.micro --iam-instance-profile Name=ecsInstanceRole --key-name aws-admin --security-group-ids sg-23b7064b --user-data file://copy-ecs-config-to-s3` -- Start 3 new EC2 instances from the default us-east-2 ESC container AMI (image), with "InstanceRole", having our SSH key installed, allowed by the security group (with specific ports open), with a user script to be executed at instance startup containing the cluster agent configuration.

 - `aws ec2 describe-instance-status --instance-id i-0c74a0c20ed35fb51` -- Don't forget the `--instance-id` (from the output of the previous command), otherwise it will return an empty result.

 - `aws ecs list-container-instances --cluster deepdive --container-instances [Arm from prev cmd output]` -- Info on container instances, including things like Docker version running, status (ACTIVE/INACTIVE), running tasks, etc.

 - `aws ec2 terminate-instances --instance-ids ...`

### Task Definitions

A *task definition* is a JSON document describing how your applications Docker Images should be ran. It's kind of like a `docker-compose` file, ala AWS. The application can include one or more Docker container instances.  It might associate volumes to the containers. It defines the Docker images, along with settings such as the memory and CPU resources assigned to a container, or the ports mapped to a container.

Example commands:

 - `aws ecs register-task-definition --cli-input-json file://web-task-definition.json`
 - `aws ecs list-task-definition-families`
 - `aws ecs describe-container-instances --cluster deepdive --container-instances arn:aws:ecs:us-east-2:514778609496:container-instance/aecd6737-cc3f-467c-8115-6a27474262c1`

### Running Tasks

A *Task* the end result of running a task definition. Running a task is comparable to a *service*, but with the task, the containers do not need to be automatically restarted when stopped (a service will try to keep tasks running). It is useful to execute one-off task, like migrations or other batch jobs.

The `run-task` command randomly distributes the tasks among the container instanes in the cluster, trying to minimize resource overload.

The [`aws start-task`](http://docs.aws.amazon.com/cli/latest/reference/ecs/start-task.html) command runs a task on a particular instance or instances (useful when certain instances are better suited for the task, e.g. have more mamory or CPUs).

Service and task life cycle (as tracked and reported by the *Container Agent*):

 - PENDING
 - RUNNING
 - STOPPED

Example commands:

 - `aws ecs run-task --cluster deepdive --task-definition web --count 1`
 - `aws ecs list-tasks --cluster deepdive`
 - `aws ecs stop-task --cluster development --task [full ARN of the task from prev cmd output]`

 - `aws ecs list-container-instances --cluster deepdive`
 - `aws ecs start-task --cluster deepdive --task-definition web --container-instances [ARN of instance from prev cmd output]`
 - `aws ecs stop-task --cluster deepdive --task [task ARN]`

### Scheduling Services

A *service* consists of a certain number of a long running task. This enables both availability as well as scalability to your application. As an example, imagine a web application, which consists of different instances of the web server running on different container instances.

A service also helps with the monitoring of existing tasks. Should one task fail or stop running due to whatever reason, a new task is automatically started up again.

The *scheduler* determines where a service or a task will run on a cluster by figuring out the most optimal instance to run it on.

A service can be "hooked up" to an *ELB* (Elastic Load Balancer) where ELB handles adding and removing instances automatically when the service scales up/down or when an instance becomes unhealthy.

The [Service Definition](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/service_definition_parameters.html) defines which task definition to use with your service, how many instantiations of that task to run, and which load balancers (if any) to associate with your tasks.

Decisions the *scheduler* will make to place a task into a cluster:

 1. Collect the service requirements (CPU, Memory, ports) and collect stats about currently running service tasks in all Container Instances in all availibility zones.
 2. Selects the best Availibility Zone (with the most remaining resources)
 3. Selects the best Container Instance within the AZ (with the most remaining resources)

Example commands:

 - `aws ecs create-service --cluster deepdive --service-name web --task-definition web --desired-count 1`
 - `aws ecs list-services --cluster deepdive`
 - `aws ecs describe-services --cluster deepdive --services web` -- Look for the `events` section when troublshooting!
 - `aws ec2 describe-instances` - to get to know the public DNS name of the service
 - `aws ecs update-service --cluster deepdive --service web --task-definition web --desired-count 2` -- More instances
 - `aws ecs update-service --cluster deepdive --service web --task-definition web --desired-count 0` -- No instances!
 - `aws ecs delete-service --cluster deepdive --service web`
 - `aws ecs list-services --cluster deepdive`
 - `aws ecs create-service --generate-cli-skeleton` -- Generates a JSON template with all possible parameters to set.

### Tearing Down a Cluster

 - `aws ec2 terminate-instances --instance-ids i-0c74a0c20ed35fb51`
 - `aws s3 rm s3://tkarakai-gto-test --recursive`
 - `aws s3api delete-bucket --bucket tkarakai-gto-test`

### Ruby on Rails App example

 - `docker run --rm --user "$(id -u):$(id -g)" -v "$PWD":/usr/src/app -w /usr/src/app rails:4 rails new --skip-bundle dockerzon`


## Glossary

 - AMI -- [Amazon Machine Image](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html)
 - ARN -- [Amazon Resource Name](http://docs.aws.amazon.com/AmazonS3/latest/dev/s3-arn-format.html)


## Tools, tips

- Web Console keeps changing, CLI is more stable, use CLI when can
- Install and use [aws-shell](https://github.com/awslabs/aws-shell)! Super useful with the CLI.

### Troubleshooting

**PROBLEM:** When you issue `aws s3api create-bucket --bucket tkarakai-gto-dockerzon` and it results in the following error:
`An error occurred (IllegalLocationConstraintException) when calling the CreateBucket operation: The unspecified location constraint is incompatible for the region specific endpoint this request was sent to.`,

**SOLUTION:** ...just add the location constraint to the command like this: `aws s3api create-bucket --bucket tkarakai-gto-dockerzon --region us-east-2 --create-bucket-configuration LocationConstraint=us-east-2`


**PROBLEM** How do I add instances to a load balancer?

**PROBLEM** How do I start tasks on a clulster?

**PROBLEM** How does the `aws ec2 run-instances --image-id ami-...` command knows to join an ECS cluster?

**SOLUTION** Based on the `ecs.config` file, in  there the `ECS_CLUSTER=production` confiuration (in this case specifying the cluster called `production`). This file is made available by copying it in to the instance from an S3 bucket. The script that copies it in is specified in the `--user-data file://copy-ecs-config-to-s3` which gets executed on the instance at startup time.

File: copy-ecs-config-to-s3
```
#!/bin/bash

yum install -y aws-cli
aws s3 cp s3://tkarakai-gto-dockerzon/ecs.config /etc/ecs/ecs.config
```
File: ecs.config
```
ECS_CLUSTER=deepdive
```
