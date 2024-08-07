# Deployed and Hosted a Highly Available WordPress Application Using AWS

This project demonstrates how to deploy and host a highly available WordPress application using AWS cloud services. The project architecture leverages multiple AWS resources to ensure high availability, scalability, and security. Below is an overview of the architecture and the steps taken to deploy the application.

## Project Overview

The primary goal of this project is to host a WordPress website on AWS, utilizing various AWS services for optimal performance and reliability. The infrastructure includes:

- **Virtual Private Cloud (VPC)**: Configured with public and private subnets across two availability zones for improved reliability and fault tolerance.
- **Internet Gateway**: Provides connectivity between VPC instances and the Internet.
- **Security Groups**: Act as network firewalls to control inbound and outbound traffic to AWS resources.
- **Availability Zones**: Two zones are used to enhance system reliability and fault tolerance.
- **Public Subnets**: Utilized for NAT Gateway and Application Load Balancer.
- **Private Subnets**: Used for web servers (EC2 instances) and secure connections via EC2 Instance Connect Endpoint.
- **NAT Gateway**: Enables private subnet instances to access the Internet.
- **EC2 Instances**: Host the WordPress application.
- **Application Load Balancer**: Distributes web traffic evenly across multiple EC2 instances.
- **Auto Scaling Group**: Automatically manages EC2 instances for availability and scalability.
- **EFS (Elastic File System)**: Provides shared storage for EC2 instances.
- **RDS (Relational Database Service)**: Manages the WordPress database.
- **Certificate Manager**: Secures application communications with SSL/TLS certificates.
- **Route 53**: Manages DNS records for domain registration.
- **Simple Notification Service (SNS)**: Sends alerts about Auto Scaling Group activities.

## Deployment Steps

Below are the steps taken to deploy the WordPress application:

1. **Configure VPC**: Set up a VPC with both public and private subnets.
2. **Deploy Internet Gateway**: Facilitate connectivity between the VPC and the Internet.
3. **Establish Security Groups**: Implement network firewalls for VPC instances.
4. **Leverage Availability Zones**: Utilize two zones for redundancy and fault tolerance.
5. **Utilize Public Subnets**: Allocate infrastructure components like NAT Gateway and Load Balancer.
6. **Implement EC2 Instance Connect Endpoint**: Securely connect to assets within subnets.
7. **Position Web Servers**: Deploy EC2 instances within private subnets for security.
8. **Enable NAT Gateway**: Allow private instances to access the Internet.
9. **Host Website**: Use EC2 instances to run WordPress.
10. **Employ Load Balancer**: Distribute traffic across EC2 instances.
11. **Utilize Auto Scaling Group**: Ensure availability and elasticity.
12. **Store Web Files on GitHub**: Manage version control and collaboration.
13. **Secure Communications**: Use Certificate Manager for SSL/TLS.
14. **Configure SNS**: Set up notifications for Auto Scaling activities.
15. **Register Domain**: Use Route 53 for DNS management.
16. **Use EFS**: Share file systems across instances.
17. **Use RDS**: Manage the database for WordPress.

## Installation Script

To install WordPress on an EC2 instance:

```bash
# Switch to root user
sudo su

# Update the software packages on the EC2 instance
sudo yum update -y

# Create an HTML directory
sudo mkdir -p /var/www/html

# Set environment variable for EFS DNS name
EFS_DNS_NAME=

# Mount the EFS to the HTML directory
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport "$EFS_DNS_NAME":/ /var/www/html

# Install and start Apache web server
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

# Install PHP 8 and necessary extensions for WordPress
sudo dnf install -y php php-cli php-cgi php-curl php-mbstring php-gd php-mysqlnd php-gettext php-json php-xml php-fpm php-intl php-zip php-bcmath php-ctype php-fileinfo php-openssl php-pdo php-tokenizer

# Install MySQL Community Repository
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf repolist enabled | grep "mysql.*-community.*"
sudo dnf install -y mysql-community-server

# Start and enable MySQL server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Set permissions
sudo usermod -a -G apache ec2-user
sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
sudo find /var/www -type f -exec sudo chmod 0664 {} \;
sudo chown apache:apache -R /var/www/html

# Download and extract WordPress files
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/

# Create the wp-config.php file
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php

# Edit the wp-config.php file
sudo vi /var/www/html/wp-config.php

# Restart the webserver
sudo service httpd restart
```

## Auto Scaling Group Launch Template Script

For setting up the Auto Scaling Group:

```bash
#!/bin/bash

# Update the software packages on the EC2 instance
sudo yum update -y

# Install and start Apache web server
sudo yum install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

# Install PHP 8 and necessary extensions for WordPress
sudo dnf install -y php php-cli php-cgi php-curl php-mbstring php-gd php-mysqlnd php-gettext php-json php-xml php-fpm php-intl php-zip php-bcmath php-ctype php-fileinfo php-openssl php-pdo php-tokenizer

# Install MySQL Community Repository
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf repolist enabled | grep "mysql.*-community.*"
sudo dnf install -y mysql-community-server

# Start and enable MySQL server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Set environment variable for EFS DNS name
EFS_DNS_NAME=

# Mount the EFS to the HTML directory
echo "$EFS_DNS_NAME:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
mount -a

# Set permissions
chown apache:apache -R /var/www/html

# Restart the webserver
sudo service httpd restart
```

## Additional Resources

For more information on the project, including screenshots and more comprehensive steps, please refer to the documentation file [Deployment Guide](./DEPLOYMENT-GUIDE.md).
