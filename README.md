# Dynamic Website Hosting on AWS

![Architecture Diagram](architecture-diagram.png)

## Project Overview

This project demonstrates the manual design and deployment of a production-style dynamic website on Amazon Web Services using a multi-tier architecture.

All infrastructure components were created manually through the AWS console and CLI.  
All scripts were executed manually over SSH.  
No EC2 user data was used.

The architecture separates networking, database, migration, and application layers and follows real-world deployment order and operational practices.

---

## Architecture Overview

The application is hosted inside a custom Amazon VPC spanning two Availability Zones for high availability and fault tolerance.

The environment is organized into distinct layers: network, data, migration, and application.

Incoming traffic is handled by an Application Load Balancer deployed in public subnets. Web servers run on EC2 instances inside private **application subnets** and are never exposed directly to the internet.

Outbound internet access for private subnets is provided by a NAT Gateway. Administrative access to private resources is handled securely using an EC2 Instance Connect Endpoint.

The database layer uses Amazon RDS (MySQL), deployed in private **data subnets** with no public access. Database access is restricted using security groups.

Database schema and seed data are migrated using Flyway from a dedicated EC2 migration server **before application deployment**.

Application code and migration assets are stored in Amazon S3.

Web servers are built from a **custom AMI** created from a manually configured EC2 instance and deployed using a Launch Template, Target Group, Application Load Balancer, and Auto Scaling Group.

DNS routing and HTTPS are configured using Amazon Route 53 and AWS Certificate Manager.

---

## AWS Services Used

- Amazon VPC
- Public Subnets
- Private Application Subnets
- Private Data Subnets
- Internet Gateway
- NAT Gateway
- EC2
- EC2 Instance Connect Endpoint
- Auto Scaling Group
- Launch Template
- Application Load Balancer
- Target Groups
- Amazon RDS (MySQL)
- Security Groups
- AWS Certificate Manager
- Amazon Route 53
- Amazon S3
- Amazon SNS

---

## Deployment Order (Important)

This project was deployed in the following order:

1. Network and subnets
2. Database subnets and RDS
3. Database migration server and Flyway migration
4. Application EC2 build
5. Custom AMI creation
6. Launch Template
7. Target Group and Load Balancer
8. Auto Scaling Group
9. DNS and SSL

---

## Setup and Configuration Steps (Actual Deployment Order)

1. Create a VPC spanning two Availability Zones  
2. Create public subnets for:
   - Application Load Balancer  
   - NAT Gateway  
3. Create private **data subnets** for the database layer  
4. Create private **application subnets** for EC2 web servers  
5. Attach an Internet Gateway to the VPC  
6. Deploy a NAT Gateway in a public subnet  
7. Configure route tables for public, application, and data subnets  
8. Create security groups for:
   - Application Load Balancer  
   - AppServer EC2 instances  
   - Data Migration EC2 instance  
   - RDS database access  
9. Create a DB subnet group using the private data subnets  
10. Provision an Amazon RDS MySQL instance (no public access)  
11. Launch a dedicated EC2 **Data Migration Server**  
12. Execute Flyway database migrations against RDS  
13. Terminate the migration EC2 instance after successful migration  
14. Launch an EC2 instance to build the web server manually  
15. Install and configure the application on the instance  
16. Create a custom AMI from the configured EC2 instance  
17. Create a Launch Template using the custom AMI  
18. Create a Target Group  
19. Create an Application Load Balancer in public subnets  
20. Register the Target Group with the Load Balancer  
21. Create an Auto Scaling Group using private application subnets  
22. Configure HTTPS using AWS Certificate Manager  
23. Configure Route 53 DNS records pointing to the Load Balancer  

---

## Database Layer (Amazon RDS)

The application uses Amazon RDS (MySQL) as its managed relational database.

The database is deployed in private data subnets and is not publicly accessible.

### Database Security Group Rules

**Inbound**
- MySQL (TCP 3306) from AppServer Security Group  
- MySQL (TCP 3306) from Data Migration Server Security Group  

**Outbound**
- All traffic to `0.0.0.0/0`

---

## Database Migration Layer

Database schema and seed data are migrated **before application deployment**.

A dedicated EC2 instance is used exclusively for database migrations and is not part of the Auto Scaling Group.

---

## Data Migration Server Security Group

**Inbound**
- SSH (TCP 22) from EC2 Instance Connect Endpoint Security Group  

**Outbound**
- All traffic to `0.0.0.0/0`

---

## Database Migration Script (Manual Execution)

Sensitive values are intentionally masked.

```bash
#!/bin/bash

S3_URI=s3://<SQL_BUCKET_NAME>/V1__shopwise.sql
RDS_ENDPOINT=<RDS_ENDPOINT>
RDS_DB_NAME=<DATABASE_NAME>
RDS_DB_USERNAME=<DB_USERNAME>
RDS_DB_PASSWORD=<DB_PASSWORD>

sudo yum update -y

sudo wget -qO- https://download.red-gate.com/maven/release/com/redgate/flyway/flyway-commandline/10.9.1/flyway-commandline-10.9.1-linux-x64.tar.gz | tar -xvz

sudo ln -s $(pwd)/flyway-10.9.1/flyway /usr/local/bin

sudo mkdir sql

sudo aws s3 cp "$S3_URI" sql/

flyway -url=jdbc:mysql://"$RDS_ENDPOINT":3306/"$RDS_DB_NAME" \
-user="$RDS_DB_USERNAME" \
-password="$RDS_DB_PASSWORD" \
-locations=filesystem:sql \
migrate
```

---

## Application EC2 Build (Manual)

A standalone EC2 instance was launched specifically to build and configure the web server.

All configuration and application setup were performed manually via SSH.  
This instance was not part of the Auto Scaling Group and was used only to prepare the final server state.

---

## Application Deployment Script (Manual Execution)

The deployment script was executed manually on the EC2 instance **before custom AMI creation**.

This ensured the AMI contained a fully configured and validated application environment prior to being used by the Launch Template and Auto Scaling Group.

---

```bash
#!/bin/bash

sudo yum update -y

sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

sudo dnf install -y \
php php-pdo php-openssl php-mbstring php-exif php-fileinfo php-xml \
php-ctype php-json php-tokenizer php-curl php-cli php-fpm php-mysqlnd \
php-bcmath php-gd php-cgi php-gettext php-intl php-zip

sudo sed -i '/<Directory "\/var\/www\/html">/,/<\/Directory>/ s/AllowOverride None/AllowOverride All/' /etc/httpd/conf/httpd.conf

S3_BUCKET_NAME=<APPLICATION_FILES_BUCKET>

sudo aws s3 sync s3://$S3_BUCKET_NAME /var/www/html

cd /var/www/html
sudo unzip shopwise.zip
sudo cp -R shopwise/. /var/www/html/
sudo rm -rf shopwise shopwise.zip

sudo chmod -R 777 /var/www/html
sudo chmod -R 777 storage/

sudo vi .env

sudo service httpd restart
```bash

## Custom AMI Creation

The configured EC2 instance was stopped and used to create a custom Amazon Machine Image (AMI).
This AMI serves as the base image for all application servers launched by the Auto Scaling Group, ensuring consistency across instances.

---

## Application Access

The application is accessed through a custom domain secured with HTTPS.

Traffic flow:

User → Route 53 → Application Load Balancer → EC2 (Private Subnets) → Amazon RDS

---

## Key Skills and Tools Demonstrated

- VPC and subnet design  
- Data subnet isolation  
- Amazon RDS security  
- Flyway database migrations  
- Dedicated migration hosts  
- Custom AMI workflows  
- Launch Templates and Auto Scaling  
- Application Load Balancer and Target Groups  
- Manual Linux server configuration  
- Bash scripting  
- IAM role-based access  
- Production-style AWS architecture  


