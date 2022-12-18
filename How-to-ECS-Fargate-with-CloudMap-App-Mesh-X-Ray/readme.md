## Prerequisites
## Step-1: Create AWS RDS Database
- Create a RDS MySQL Database
- **Chose a Database Creation Method**
    - Standard Create
- **Select Engine**
    - Engine Options: MySQL
- **Template**
    - Only enable options eligible for RDS Free Usage Tier: Free tier
- **Settings**
    - DB Instance Identifier: microservicesdb
    - Master Username: dbadmin
    - Master Password: **supersecretpassword**
    - Confirm Password: **supersecretpassword**
- **DB Instance Size**
    - leave defaults
- **Storage**    
    - Uncheck Enable storage autoscaling for safety purpose as this is test db.
    - rest all leave to defaults
- **Connectivity**
    - VPC: ecs-vpc
    - Subnet Group: Create new DB subnet group    
    - Publicly Accessible: Yes (For troubleshooting purposes - Not required in real cases)
    - VPC Security Group
        - Create new VPC Secutiry Group: check
        - Name of VPC Security Group: microservices-rds-db-sg
    - Availability Zone: ap-south-1a
    - Database Port: 3306 (leave default)       
- **Database Authentication**
    - Leave defaults
- **Additional Configuration**
    - Database Options
        - Initial Database Name: usermgmt
    - Backup
        - Uncheck
    - rest all default        
## Step-2: Create Environment Variables using template below
- Gather the following details from RDS MySQL database to provide them as environment variables in our ECS Task Definition
```
AWS_RDS_HOSTNAME = microservicesdb.cxojydmxwly6.us-east-1.rds.amazonaws.com
AWS_RDS_PORT = 3306
AWS_RDS_DB_NAME = usermgmt
AWS_RDS_USERNAME = dbadmin
AWS_RDS_PASSWORD = supersecretpassword
```
## Step-3: Create Simple Email Service - SES SMTP Credentials
### SMTP Credentials
- SMTP Settings --> Create My SMTP Credentials
- **IAM User Name:** append the default generated name with microservice or something so we have a reference of this IAM user created for our ECS Microservice deployment
- Download the credentials and update the same for below environment variables which you are going to provide in container definition section of Task Definition. 
```
AWS_MAIL_SERVER_HOST = email-smtp.us-east-1.amazonaws.com
AWS_MAIL_SERVER_USERNAME =
AWS_MAIL_SERVER_PASSWORD =
AWS_MAIL_SERVER_FROM_ADDRESS = use-a-valid-email@gmail.com 
```
### Verfiy Email Addresses to which notifications we need to send.
- We need two email addresses for testing Notification Service.  
-  **Email Addresses**
    - Verify a New Email Address
    - Email Address Verification Request will be sent to that address, click on link to verify your email. 
    - From Address: abcd@gmail.com (replace with your ids during verification)
    - To Address: wxyz@gmail.com (replace with your ids during verification)
- **Important Note:** We need to ensure all the emails (FromAddress email) and (ToAddress emails) to be verified here. 
    - Reference Link: https://docs.aws.amazon.com/ses/latest/DeveloperGuide/verify-email-addresses.html    

## Step-4: Create Application Load Balancer
- **Create Application Load Balancer**
    - Name: microservices-alb
    - Availability Zones: ecs-vpc:  us-east-1a, us-east-1b
    - Security Group: microservices-sg-alb (Inbound: Port 80)
    - Target Group: temp-tg-micro (Rest all defaults)

