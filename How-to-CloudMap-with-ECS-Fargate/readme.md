# Microservices Deployment on AWS Fargate & ECS Clusters

### Micro service architecture 
- We are going to deploy two microservices.
    - User Management Service
    - Notification Service

### Usecase Description
- User Management **Create User API**  will call Notification service **Send Notification API** to send an email to user when we create a user. 

### List of Docker Images used in this section
| Application Name                 | Docker Image Name                          |
| ------------------------------- | --------------------------------------------- |
| User Management Microservice | stacksimplify/usermanagement-microservice:1.0.0 |
| Notifications Microservice | stacksimplify/notifications-microservice:1.0.0 |

### Architecture without service discovery
 <img width="1434" alt="ArchitectureImage" src="https://user-images.githubusercontent.com/87567444/207402503-ced0cdba-126f-4837-ad30-88e82c5247db.png">

### Architecture with service discovery
<img width="1429" alt="ArchitectureImage" src="https://user-images.githubusercontent.com/87567444/207403694-65166263-d8fd-4bdb-b63a-9101ba89eb5f.png">

