# Dynamic Website Hosting on AWS

!![Architecture Diagram](Host_a_Dynamic_Web_App_on_AWS.png)

## Project Overview

This project demonstrates the design, deployment, and operation of a production-style dynamic website on Amazon Web Services. The goal was not just to host a website, but to build a secure, scalable, and fault-tolerant cloud architecture that reflects real-world infrastructure patterns.

The application runs on private compute resources, scales automatically, uses load balancing, and is deployed through a scripted automation process rather than manual configuration.

---

## Architecture Overview

The application is hosted inside a custom Amazon VPC spanning two Availability Zones for high availability.

Incoming traffic is handled by an Application Load Balancer deployed in public subnets. Web servers run on EC2 instances inside private subnets and are never exposed directly to the internet.

Outbound internet access for private instances is provided by a NAT Gateway. Administrative access is handled securely using an EC2 Instance Connect Endpoint.

DNS routing and HTTPS are configured using Route 53 and AWS Certificate Manager.

Application code and assets are stored in Amazon S3 and deployed onto instances using an automated deployment script.

---

## AWS Services Used

- Amazon VPC  
- Public and Private Subnets  
- Internet Gateway  
- NAT Gateway  
- EC2  
- Auto Scaling Group  
- Application Load Balancer  
- Security Groups  
- EC2 Instance Connect Endpoint  
- AWS Certificate Manager  
- Amazon Route 53  
- Amazon S3  
- Amazon SNS  

---

## Deployment Automation (Bash Script)

This project uses a Bash deployment script to configure EC2 instances at launch.  
This script is **not user data** logic documentation — it represents a real deployment process used to prepare application servers automatically.

### What the Script Does

- Updates system packages  
- Installs and enables Apache  
- Installs PHP and required extensions  
- Installs and enables MySQL 8  
- Configures Apache to allow application routing  
- Pulls application files from an S3 bucket  
- Extracts and deploys the application into `/var/www/html`  
- Sets required file permissions  
- Prepares the application environment configuration  
- Restarts web services  

This ensures every new instance launched by Auto Scaling becomes a fully functional web server without manual intervention.

---

## Deployment Script

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

sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf install -y mysql-community-server
sudo systemctl start mysqld
sudo systemctl enable mysqld

sudo sed -i '/<Directory "\/var\/www\/html">/,/<\/Directory>/ s/AllowOverride None/AllowOverride All/' /etc/httpd/conf/httpd.conf

S3_BUCKET_NAME=joshcloud-project-web-files
sudo aws s3 sync s3://joshcloud-project-web-files/shopwise.zip /var/www/html

cd /var/www/html
sudo unzip shopwise.zip
sudo cp -R shopwise/. /var/www/html/
sudo rm -rf shopwise shopwise.zip

sudo chmod -R 777 /var/www/html
sudo chmod -R 777 storage/

sudo vi .env

sudo service httpd restart
---

## Setup and Configuration Steps

1. Create a VPC with public and private subnets across two Availability Zones  
2. Attach an Internet Gateway to enable public internet access  
3. Deploy a NAT Gateway in a public subnet for outbound traffic from private instances  
4. Configure route tables for public and private subnet traffic flow  
5. Create security groups for:
   - Application Load Balancer
   - EC2 instances
   - Database access
6. Create an Application Load Balancer in the public subnets  
7. Configure an Auto Scaling Group using a Launch Template  
8. Upload application files to an Amazon S3 bucket  
9. Attach an IAM role to EC2 instances with S3 read permissions  
10. Configure Route 53 DNS records and SSL certificates using AWS Certificate Manager  

---

## Application Access

The application is accessed through a custom domain secured with HTTPS.

Traffic flow:

User → Route 53 → Application Load Balancer → EC2 (Private Subnets)


---

## Key Skills and Tools Demonstrated

- AWS VPC and network architecture  
- High availability and fault-tolerant system design  
- Auto Scaling and load balancing  
- Secure private subnet deployments  
- Linux server administration  
- Bash scripting and deployment automation  
- IAM role-based access control  
- DNS and SSL configuration  
- Production-style cloud infrastructure practices  

---

## License

This project is licensed under the MIT License.
