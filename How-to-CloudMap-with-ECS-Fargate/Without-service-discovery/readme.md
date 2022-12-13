# Microservices Deployment on AWS Fargate & ECS Clusters

We are going to implement the above environment
<img width="1434" alt="ArchitectureImage" src="https://user-images.githubusercontent.com/87567444/207402503-ced0cdba-126f-4837-ad30-88e82c5247db.png">

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
