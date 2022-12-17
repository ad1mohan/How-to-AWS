# Microservices Deployment on AWS Fargate & ECS Clusters

We are going to implement the above environment
<img width="1434" alt="ArchitectureImage" src="https://user-images.githubusercontent.com/87567444/207402503-ced0cdba-126f-4837-ad30-88e82c5247db.png">

# Module - 1: Deploy Notification Microservice
## Step-1: Notification Microservice - Create Task Definition
- **Configure Task Definition**
    - Task Definition Name: notification-microservice-td
    - Task Role: ecsTaskExecutionRole
    - Network Mode: awsvpc
    - Task Execution Role: ecsTaskExecutionRole
    - Task Memory: 2GB
    - Task CPU: 1 vCPU
- **Configure Container Definition**    
    - Container Name: notification-microservice
    - Image: stacksimplify/notifications-microservice:1.0.0
    - Container Port: 8096
    - Environment Variables
        - AWS_MAIL_SERVER_HOST=email-smtp.us-east-1.amazonaws.com
        - AWS_MAIL_SERVER_USERNAME=*****
        - AWS_MAIL_SERVER_PASSWORD=*****
        - AWS_MAIL_SERVER_FROM_ADDRESS=stacksimplify@gmail.com

## Step-2: Notification Microservice - Create Service
- **Configure Service**
    - Launch Type: Fargate
    - Task Definition:
        - Family: notification-microservice-td
        - Version: 1(latest) 
    - Service Name: svc-notification-microservice
    - Number of Tasks: 1
- **Configure Network**
    - VPC: ecs-vpc
    - Subnets: us-east-1a, us-east-1b
    - Security Groups: ecs-microservices-inbound 
        - Allow traffic from ALB security group or anywhere
        - Inbound Port 8095 and 8096 (for both notification & usermanagement service)
        - Notification Service uses Port 8096
        - User Management Service uses port 8095
    - AutoAssign Public IP: Enabled                
    - Health Check Grace Period: 147
    - **Load Balancing**
        - Load Balancer Type: Application Load Balancer
        - Load Balancer Name: microservices-alb        
        - Container to Load Balance: notification-microservice : 8096  **Click Add to Load Balancer **
        - Target Group Name: tg-notification-microservice
        - Path Pattern: /notification*
        - Evaluation Order: 1
        - Health Check path: /notification/health-status
- **Important Note:** Disable Service Discovery for now

## Step-3: Notification Microservice - Verify the deployment and access the service. 
- Verify tasks are in **RUNNING** state in Service
- Verify ALB - Target Groups section, Registered Tagets status to confirm if health checks are succeeding. If healthy, you shoud see
    - Status: Healthy
    - Description: This target is currently passing target group's health checks.
- **Troubleshooting Tip**
    - If we get **Request Timed out** as status, cross check ECS security group assigned to **svc-notificaiton-microservice** whether port 8096 is open for inbound traffic. 
- Verify using Load Balancer URL or DNS registered URL
```
http://services.stacksimplify.com/notification/health-status
```

# Module - 2: Deploy User Management Microservice
## Step-1: User Management Service - Create Task Definition
- **Configure Task Definition**
    - Task Definition Name: usermgmt-microservice-td1
    - Task Role: ecsTaskExecutionRole
    - Network Mode: awsvpc
    - Task Execution Role: ecsTaskExecutionRole
    - Task Memory: 2GB
    - Task CPU: 1 vCPU
- **Configure Container Definition**
    - Container Name: usermanagement-microservice
    - Image: stacksimplify/usermanagement-microservice:1.0.0
    - Container Port: 8095
    - **Environment Variables**
        - AWS_RDS_HOSTNAME=microservicesdb.cxojydmxwly6.us-east-1.rds.amazonaws.com
        - AWS_RDS_PORT=3306
        - AWS_RDS_DB_NAME=usermgmt
        - AWS_RDS_USERNAME=dbadmin
        - AWS_RDS_PASSWORD=*****
        - NOTIFICATION_SERVICE_HOST=services.stacksimplify.com [or] ALB DNS Name
        - NOTIFICATION_SERVICE_PORT=80

## Step-2: Allow Connections from ECS to RDS DB
- Identify the ECS security group which we are going to use for User Management Service
- Update the RDS Database security group inbound rules to allow connectivity for port 3306 from ECS Usermanagement security group.

## Step-3: User Management Service - Create Service
- **Configure Service**
    - Launch Type: Fargate
    - Task Definition:
        - Family: usermgmt-microservice-td1
        - Version: 1(latest) 
    - Service Name: svc-usermgmt-microservice
    - Number of Tasks: 1
- **Configure Network**
    - VPC: ecs-vpc
    - Subnets: us-east-1a, us-east-1b
    - Security Groups: ecs-microservices-inbound (already configured during notification service)
        - Allow traffic from ALB security group or anywhere
        - Inbound Port 8095 and 8096 (for both notification & usermanagement service)
    - AutoAssign Public IP: Enabled                
    - Health Check Grace Period: 147
    - **Load Balancing**
        - Load Balancer Type: Application Load Balancer
        - Load Balancer Name: microservices-alb        
        - Container to Load Balance: usermgmt-microservice : 8095  **Click Add to Load Balancer **
        - Target Group Name: tg-usermgmt-microservice
        - Path Pattern: /usermgmt*
        - Evaluation Order: 1
        - Health Check path: /usermgmt/health-status
- **Important Note:** Disable Service Discovery for now


## Step-4: User Management Microservice - Verify the deployment and access the service. 
- Verify tasks are in **RUNNING** state in Service
- Verify ALB - Target Groups section, Registered Tagets status to confirm if health checks are succeeding. If healthy, you shoud see
    - Status: Healthy
    - Description: This target is currently passing target group's health checks.
- **Troubleshooting Tip**
    - If we get **Request Timed out** as status, cross check ECS security group assigned to **svc-notificaiton-microservice** whether port 8096 is open for inbound traffic. 
- Verify using Load Balancer URL or DNS registered URL
```
http://services.stacksimplify.com/usermgmt/health-status
```
