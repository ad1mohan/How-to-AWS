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
        - AWS_MAIL_SERVER_FROM_ADDRESS=abc@gmail.com
## Step-2: Notification Microservice - Create Service with Service Discovery enabled
- **Configure Service**
    - Launch Type: Fargate
    - Task Definition:
        - Family: notification-microservice-td
        - Version: 1(latest) 
    - Service Name: appmesh-notification
    - Number of Tasks: 1
- **Configure Network**
    - VPC: ecs-vpc
    - Subnets: ap-south-1a, ap-south-1b
    - Security Groups: appmesh-ecs-inbound 
        - Inbound Ports 8096, 8095    
    - AutoAssign Public IP: Enabled        
- **Load Balancing**
    - Load Balancer Type: None
    - **Important Note:** As we are using Service Discovery DNS name for Notification Service, this service doesnt need load balancing via ALB. 
- **Service Discovery**
    - Enable service discovery integration: Checked
    - Namespace: create new private name space 
    - Namespace Name: stacksimplify-dev.com
    - Configure service discovery service: Create new service discovery service
    - Service discovery name: notification-service
    - Enable ECS task health propagation: checked
    - Rest all defaults
- Verify in AWS Cloud Map about the newly created Namespace, Service and registered Service Instance for Notification Microservice. 

## Important Note: Before moving to User Management Service
- In Step-3, during the notification service creation we have created a new Security Group for ECS named **appmesh-ecs-inbound**.  We need to ensure that traffic from this security group is allowed on RDS Database. 
- User Management Service when depployed it first checks the connectivity to RDS Database and if we dont update RDS Database Security group to allow traffic from **appmesh-ecs-inbound** then we are going to have an issue with UMS Service (It keeps always restarting)
- Update the RDS Security Group
    - **Name: microservices-rds-db-sg**
    - New Rule: 
        - Type: MYSQL/Aurora
        - Protocol: TCP
        - Port Range: 3306
        - Source: appmesh-ecs-inbound
        - Description: Allow access from new ECS security group named appmesh-ecs-inbound to RDS Database. 
## Step-3: User Management Service - Create Task Definition
- **Configure Task Definition**
    - Task Definition Name: usermgmt-microservice-td
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
        - NOTIFICATION_SERVICE_HOST=notification-service.stacksimplify-dev.com
        - NOTIFICATION_SERVICE_PORT=8096
## Step-4: User Management Microservice - Create Service with Service Discovery enabled
- **Configure Service**
    - Launch Type: Fargate
    - Task Definition:
        - Family: usermanagement-microservice-td
        - Version: 2(latest) 
    - Service Name: appmesh-usermanagement
    - Number of Tasks: 1
- **Configure Network**
    - VPC: ecs-vpc
    - Subnets: ap-south-1a, ap-south-1b
    - Security Groups: appmesh-ecs-inbound
        - Inbound Port 8095, 8096     
    - AutoAssign Public IP: Enabled        
- **Load Balancing**
    - Load Balancer Type: Application Load Balancer
    - Health Check Grace Period: 147
    - Load Balancer: microservices-appmesh-alb
    - Production Listener Port: 80:HTTP
    - Target Group Name:  Create New:  appmesh-usermgmt
    - Path Pattern: /usermgmt*
    - Evaluation Order: 1
    - Health Check Path: /usermgmt/health-status
- **Service Discovery**
    - Enable service discovery integration: Checked
    - Namespace: create new private name space 
    - Namespace Name: stacksimplify-dev.com
    - Configure service discovery service: Create new service discovery service
    - Service discovery name: usermanagement-service
    - Enable ECS task health propagation: checked
    - Rest all defaults
- Verify in AWS Cloud Map about the newly created Namespace, Service and registered Service Instance for User Management Microservice. 
- Update User Management Service Task Definition with Notification Service Discovery name
    - notification-service.stacksimplify-dev.com
- Update the **appmesh-usermanagement** with latest task defintion. 
- **Testing using Postman**
    - List Users
    - Delete User
    - Create User
    - Notification Info Service

## Step-5: AWS App Mesh: Create Mesh
- **Create Mesh**
    - Name: microservices-mesh
    - **Egress filter:** Allow External Traffic
- **Microservices Service Discovery Names**
    - **User Management Service:** usermanagement-service.stacksimplify-dev.com
    - **Notification Service:** notification-service.stacksimplify-dev.com

## Step-6: Notification Microservice - Create Virtual Node & Virtual Service
- **Virtual Node:**
    - Virtual Node Name: notification-vnode
    - Service Discovery Method: DNS
    - DNS Hostname: notification-service.stacksimplify-dev.com
    - Backend: Nothing
    - Health Status: Nothing
    - Listener Port: 8096 Protocol: HTTP
    - Health Check: Disabled
- **Virtual Service:**    
    - Virtul Service Name: notification-service.stacksimplify-dev.com
    - Provider: Virtual Node
    - Virtual Node: notification-vnode

## Step-7: User Management Microservice - Create Virtual Node & Virtual Service
- **Virtual Node:**
    - Virtual Node Name:usermgmt-vnode
    - Service Discovery Method: DNS
    - DNS Hostname: usermanagement-service.stacksimplify-dev.com
    - Backend: Nothing
    - Health Status: Nothing
    - Listener Port: 8095 Protocol: HTTP
    - Health Check: Disabled
- **Virtual Service:**    
    - Virtul Service Name: usermanagement-service.stacksimplify-dev.com
    - Provider: Virtual Node
    - Virtual Node: usermgmt-vnode

## Step-8: Update backend for Virtual Node: usermgmt-vnode
- Service Backends: notification-service.stacksimplify-dev.com

## Step-9: IAM Changes - Provide access for ecsTaskExecutionRole
- **Role Name:** ecsTaskExecutionRole
- Navigate to IAM and update this role by attaching below listed policy
    - **Policy Name:** AWSXRayDaemonWriteAccess

## Step-10: Task Definition Update: Notification Microservice to Enable AppMesh & X-Ray
- **X-Ray Container**
    - Container Name: xray-daemon
    - Image: amazon/aws-xray-daemon:1
    - Port Mappings: 2000 Protocol: UDP
    - Log Configuration: Enabled
- **Service Integration**
    - Enable App Mesh Integration: Checked
    - Mesh name: microservices-mesh
    - AppMesh endpoints: Vitual Node
    - Application container name: notification-microservice
    - Virtual node name: notification-vnode
    - Virtual node port: 8096
- **Envoy Container Special Settings**
    - ENVOY_LOG_LEVEL: trace
    - ENABLE_ENVOY_XRAY_TRACING: 1
- **Proxy Congigurations**
    - Egress ignored Ports: 587 **VERY IMPORTANT for SES COMMUNICATION**
    - **Reference-1:** https://github.com/aws/aws-app-mesh-roadmap/issues/62
    - **Reference-2:** https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_ProxyConfiguration.html

## Step-11: Task Definition Update: User Management Microservice to Enable AppMesh & X-Ray
- **X-Ray Container**
    - Container Name: xray-daemon
    - Image: amazon/aws-xray-daemon:1
    - Port Mappings: 2000 Protocol: UDP
    - Log Configuration: Enabled
- **Service Integration**
    - Enable App Mesh Integration: Checked
    - Mesh name: microservices-mesh
    - AppMesh endpoints: Vitual Node
    - Application container name: usermanagement-microservice
    - Virtual node name: usermgmt-vnode
    - Virtual node port: 8095
- **Envoy Container Special Settings**
    - ENVOY_LOG_LEVEL: trace
    - ENABLE_ENVOY_XRAY_TRACING: 1
- **Proxy Congigurations**
    - Egress ignored Ports: 3306 **VERY IMPORTANT for MYSQL COMMUNICATION**
    - **Reference-1:** https://github.com/aws/aws-app-mesh-roadmap/issues/62
    - **Reference-2:** https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_ProxyConfiguration.html

## Step-12: Update User Management Service with new Task Definition version (AppMesh Enablement)
- Service Name: appmesh-usermanagement

## Step-13: Update Notification Service with new Task Definition version (AppMesh Enablement)
- Service Name: appmesh-notification

## Step-14: Testing using Postman
- List Users
- Delete User
- Create User
- Notification Info Service
