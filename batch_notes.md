# Batch Study notes

## Chapter 1: Architecture and Components
### Objectives
- What is batch and it's use cases
- Batch components
  - Job
  - Job Definition
  - Job Queue
  - Compute Environment
### What is Batch Processing
In the simplest terms, a batch job is a scheduled program that is assigned to run on a computer without further user interaction.
### Batch Processing use cases
- HPC
- Financial Risk Management
- Fraud Surveillance
- Drug screening
- DNA sequencing
- Digital Media
### What is AWS Batch
AWS Batch is a set of batch management capabilities that enables developers, scientists, and engineers to easily and efficiently run hundreds of thousands of batch computing jobs on AWS. AWS Batch dynamically provisions the optimal quantity and type of compute resources (e.g., CPU or memory optimized compute resources) based on the volume and specific resource requirements of the batch jobs submitted. With AWS Batch, there is no need to install and manage batch computing software or server clusters, allowing you to instead focus on analyzing results and solving problems. AWS Batch plans, schedules, and executes your batch computing workloads using Amazon ECS, Amazon EKS, and AWS Fargate with an option to utilize spot instances.
### AWS Batch uses ECS
AWS Batch uses Amazon ECS to execute containerized jobs and therefore requires the ECS Agent to be installed on compute resources within your AWS Batch Compute Environments. The ECS Agent is pre-installed in Managed Compute Environments.

[Batch FAQ's answer most of the quesitons](https://aws.amazon.com/batch/faqs/)

### Batch Componenets

#### Compute Environment
- Compute environments contain the Amazon ECS container instances that are used to run containerized batch jobs. 
- A specific compute environment can also be mapped to one or more than one job queue.
- A compute environment comprises the following: 
  - ECS Cluster
  - AMI (which by default uses ECS-Optmized)
  - EC2 instances 
    - OnDemand 
    - Spot
  - Fargate or Fargate Spot
  - IAM Instance Role
  - VPC
  - Subnet and Security Groups
  - launch template
- CE are of two types
  - MANAGED CE
    - where AWS scales and configures these resources for customers
  - UNMANAGED CE
    - wherein AWS creates the ECS Cluster and customers need to manage and scale the resources yourself
  - Fargate and Fargate Spot CE 
    - where it can run containers without having to manage servers or clusters of Amazon EC2 instances
    - customers no longer have to provision, configure, or scale clusters of virtual machines to run containers.
    - removes the need to choose server types, decide when to scale customer's clusters, or optimize cluster packing.
#### Job Defination

#### Job Queue
- Jobs are submitted to a job queue where they reside until they can be scheduled to run in a compute environment. 
- An AWS account can have multiple job queues. 
  - For example, you can create a queue that uses Amazon EC2 On-Demand instances for high priority jobs and another queue that uses Amazon EC2 Spot Instances for low-priority jobs. 
- Job queues have a priority that's used by the scheduler to determine which jobs in which queue should be evaluated for execution first.
#### Job
- An AWS Batch Job is the unit of work, which is essentially a container that will run in your AWS Batch Compute Resources based on the configurations used in the Job Definition.
- During the lifecycle of a Job, some properties and behaviors associated with the job are:
  - [Job States](https://docs.aws.amazon.com/batch/latest/userguide/job_states.html)
  - Automated Retry
  - Job Dependencies
  - Job Timeouts
  - Array Jobs
